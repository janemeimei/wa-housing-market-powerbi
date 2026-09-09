# Power Query (M) 版 — 不用 DAX 也能建出同一个模型

**用途：** 如果租户里 `New measure` 不可用，用 Power Query 做等价的聚合。
结果的数字完全一样，区别是它在**刷新时**算好（静态），DAX 度量是**每次筛选都重算**（响应上下文）。

🔴 **走这条路的话，简历上只能写 `Power Query modeling`，要去掉 `DAX measures`。**

---

## 最快的做法：Advanced Editor 整段粘贴

Power Query 编辑器 → 顶部 **`Home` → `Advanced Editor`** → 全选删掉 → 粘贴下面的代码。

### 查询 1 · `County Month`（主表，改名成这个）

```m
let
    Source = Csv.Document(
        Web.Contents("https://raw.githubusercontent.com/janemeimei/wa-housing-market-powerbi/main/data/wa_housing_county_month.csv"),
        [Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]
    ),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"Date", type date},
        {"County", type text},
        {"Year", Int64.Type},
        {"Month", Int64.Type},
        {"MonthName", type text},
        {"HomesSold", Int64.Type},
        {"Inventory", type number},
        {"NewListings", type number},
        {"PendingSales", type number},
        {"MonthsOfSupply", type number},
        {"MedianDOM", type number},
        {"PriceDropShare", type number},
        {"SoldAboveListShare", type number},
        {"SaleToListRatio", type number},
        {"MedianSalePrice", type number},
        {"MedianListPrice", type number},
        {"MedianPPSF", type number}
    }),

    // 降价分档 —— 第 2 页的分档条形图用
    WithBand = Table.AddColumn(Typed, "Price Drop Band", each
        if [PriceDropShare] = null then "(not reported)"
        else if [PriceDropShare] < 0.10 then "<10%"
        else if [PriceDropShare] < 0.20 then "10-20%"
        else if [PriceDropShare] < 0.30 then "20-30%"
        else if [PriceDropShare] < 0.40 then "30-40%"
        else "40%+", type text),

    // 散点样本标记：2024 起 · 月成交 >= 20 · 两个字段都有值
    WithSample = Table.AddColumn(WithBand, "In Scatter Sample", each
        [HomesSold] >= 20
        and [Date] >= #date(2024, 1, 1)
        and [PriceDropShare] <> null
        and [MedianDOM] <> null, type logical),

    // 加权用的中间量：成交量 x 高于挂牌价占比
    WithAboveListNum = Table.AddColumn(WithSample, "AboveListTimesSold", each
        (if [SoldAboveListShare] = null then 0 else [SoldAboveListShare]) * [HomesSold], type number),

    // 小县标记（月成交 < 20）
    WithSmallFlag = Table.AddColumn(WithAboveListNum, "SmallCountySales", each
        if [HomesSold] < 20 then [HomesSold] else 0, Int64.Type)
in
    WithSmallFlag
```

---

### 查询 2 · `State Monthly`（州级汇总 —— 第 1 页全靠它）

新建空查询：左边查询列表右键 → **`New query` → `Blank query`** → Advanced Editor → 粘贴：

```m
let
    Source = #"County Month",

    Grouped = Table.Group(Source, {"Date"}, {
        {"HomesSold",        each List.Sum([HomesSold]),                     type number},
        {"Inventory",        each List.Sum([Inventory]),                     type number},
        {"MoS_Unweighted",   each List.Average([MonthsOfSupply]),            type number},
        {"AboveListNum",     each List.Sum([AboveListTimesSold]),            type number},
        {"SmallCountySales", each List.Sum([SmallCountySales]),              type number},
        {"CountiesTotal",    each Table.RowCount(_),                         Int64.Type},
        {"CountiesReporting",each List.Count(List.Select([HomesSold], (x) => x >= 20)), Int64.Type}
    }),

    // ⭐ 加权口径 = 全州存量 ÷ 全州成交
    MoSWeighted = Table.AddColumn(Grouped, "MoS_Weighted", each
        [Inventory] / [HomesSold], type number),

    Gap = Table.AddColumn(MoSWeighted, "WeightingGap", each
        [MoS_Unweighted] - [MoS_Weighted], type number),

    AboveListW = Table.AddColumn(Gap, "AboveList_Weighted", each
        [AboveListNum] / [HomesSold], Percentage.Type),

    SmallShare = Table.AddColumn(AboveListW, "SmallCountyShareOfSales", each
        [SmallCountySales] / [HomesSold], Percentage.Type),

    SmallWeight = Table.AddColumn(SmallShare, "SmallCountyWeightInAverage", each
        ([CountiesTotal] - [CountiesReporting]) / [CountiesTotal], Percentage.Type),

    // ⭐ 两种口径各自的市场判定 —— 同一个月会给出相反答案
    VerdictU = Table.AddColumn(SmallWeight, "Verdict_Unweighted", each
        if [MoS_Unweighted] < 4 then "Seller's market"
        else if [MoS_Unweighted] <= 6 then "Balanced"
        else "Buyer's market", type text),

    VerdictW = Table.AddColumn(VerdictU, "Verdict_Weighted", each
        if [MoS_Weighted] < 4 then "Seller's market"
        else if [MoS_Weighted] <= 6 then "Balanced"
        else "Buyer's market", type text),

    Sorted = Table.Sort(VerdictW, {{"Date", Order.Ascending}})
in
    Sorted
```

---

### 查询 3 · `Seasonality Check`（证明不是季节性）

```m
let
    Source = Table.SelectRows(#"County Month", each [In Scatter Sample] = true),

    Grouped = Table.Group(Source, {"Month", "MonthName"}, {
        {"N",         each Table.RowCount(_),                              Int64.Type},
        {"MedianDOM", each List.Median([MedianDOM]),                       type number},
        {"HighDropDOM", each
            List.Average(Table.SelectRows(_, each [PriceDropShare] >= 0.3)[MedianDOM]), type number},
        {"LowDropDOM",  each
            List.Average(Table.SelectRows(_, each [PriceDropShare] < 0.3)[MedianDOM]),  type number}
    }),

    // 每个日历月内部：高降价县 减 低降价县 的在售天数
    // 12 个月应该全部为负 —— 说明不是夏天效应
    DomGap = Table.AddColumn(Grouped, "DOM Gap Within Month", each
        [HighDropDOM] - [LowDropDOM], type number),

    Sorted = Table.Sort(DomGap, {{"Month", Order.Ascending}})
in
    Sorted
```

---

## 🔴 建完必须核对（筛到 2026-05）

**这些是我在 Python 里独立算的真值。对不上就是 M 写错了。**

| 字段 | 应该等于 |
|---|---|
| `HomesSold` | **7,884** |
| `Inventory` | **24,287** |
| **`MoS_Unweighted`** | **6.62** |
| **`MoS_Weighted`** | **3.08** |
| `WeightingGap` | **3.54** |
| **`Verdict_Unweighted`** | **Buyer's market** |
| **`Verdict_Weighted`** | **Seller's market** |
| `SmallCountyShareOfSales` | **0.62%** |
| `CountiesReporting` | **30**（全州 39） |
| `SmallCountyWeightInAverage` | **23%** |
| `AboveList_Weighted` | **30.9%** |

**`Seasonality Check` 表：`DOM Gap Within Month` 这一列 12 行应该全是负数。**

---

## 如果找不到 Advanced Editor —— 纯点击版

**查询 2 的 Group by 用 UI 做：**

1. 选中 `Date` 列 → `Transform` → **`Group by`** → 选 **Advanced**
2. Group by: `Date`
3. 逐个加聚合：

| 新列名 | Operation | Column |
|---|---|---|
| `HomesSold` | Sum | HomesSold |
| `Inventory` | Sum | Inventory |
| `MoS_Unweighted` | Average | MonthsOfSupply |

4. `Add Column` → **`Custom Column`**，逐个加：

```
MoS_Weighted            =  [Inventory] / [HomesSold]

WeightingGap            =  [MoS_Unweighted] - [MoS_Weighted]

Verdict_Weighted        =  if [MoS_Weighted] < 4 then "Seller's market"
                           else if [MoS_Weighted] <= 6 then "Balanced"
                           else "Buyer's market"

Verdict_Unweighted      =  if [MoS_Unweighted] < 4 then "Seller's market"
                           else if [MoS_Unweighted] <= 6 then "Balanced"
                           else "Buyer's market"
```

⚠️ **Custom Column 里的公式区分大小写，列名要用方括号。**

---

## M 和 DAX 的差别（面试可能问）

| | Power Query (M) | DAX |
|---|---|---|
| 什么时候算 | **刷新时**，算一次 | **每次交互**都重算 |
| 结果 | 存下来的表/列 | 响应筛选上下文的值 |
| 适合 | 清洗、整形、固定粒度的汇总 | 随切片器变化的指标 |
| 这个项目 | 州级月度汇总是固定的 → M 够用 | 若要按县切片看加权值 → 必须 DAX |

⭐ **诚实说法：** "I built the state-level aggregation in Power Query because the grain is fixed.
A measure would be the better choice if the report needed to re-aggregate under a slicer."
