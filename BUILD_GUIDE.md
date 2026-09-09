# Power BI 构建指南 — 全程在浏览器里做，不需要 Windows

**为什么这么做：** 你的 Mac 只剩 27.6 GB，Windows 虚拟机要 40 GB。
Power BI Service（网页版）现在支持完整的建模 + Power Query + DAX + 报表编辑，够用。

**已验证的官方依据：**
- 网页建模：[Edit semantic models in the Power BI service](https://learn.microsoft.com/en-us/power-bi/transform-model/service-edit-data-models) — 支持 Power Query 编辑、关系管理、DAX 度量
- Free 许可：[Features by license type](https://learn.microsoft.com/en-us/power-bi/fundamentals/service-features-license-type) — Fabric (free) 可以「access content they create for themselves」，即 My Workspace 里自建自用

🔴 **一条关键限制，决定了下面的做法：**
> "You can't download reports based on local Excel or CSV files that were uploaded to Power BI."
> — [Download a report from the Power BI service](https://learn.microsoft.com/en-us/power-bi/create-reports/service-export-to-pbix)

**直接上传 CSV → 拿不到 `.pbix` 文件 → GitHub 上就没有作品可放。**
所以数据已经放在 GitHub 上，用 **Web 连接器**读取，绕开这条限制。

数据地址（已验证 HTTP 200）：
```
https://raw.githubusercontent.com/janemeimei/wa-housing-market-powerbi/main/data/wa_housing_county_month.csv
```

---

## 第一步 · 登录

1. 打开 `app.powerbi.com`
2. 🔴 **用 `@northeastern.edu` 登录** —— gmail 会被拒
3. 如果提示注册，选 **Sign up free**

---

## 第二步 · 连数据

```
左侧 Workspaces → My workspace → 右上 + New → 更多 → Semantic model
或：左侧 Create → Get data
```

1. 连接器里搜 **Web**
2. URL 粘贴上面那个 raw 地址
3. 点 **Next** → 进入 Power Query 预览

### Power Query 里要做的三件事

**① 确认第一行是表头**
如果列名显示成 `Column1 Column2`，点 **Use first row as headers**

**② 改数据类型**（点列名左边的图标改）

| 列 | 类型 |
|---|---|
| `Date` | Date |
| `County` · `MonthName` | Text |
| `Year` · `Month` · `HomesSold` · `Inventory` · `NewListings` · `PendingSales` | Whole number |
| 其余全部 | Decimal number |

🔴 **`PriceDropShare` 和 `SoldAboveListShare` 是小数（0.31 = 31%），不要改成百分比类型**，后面在视觉对象里格式化。

**③ 不需要加 Month 列** —— 数据里已经有 `Year` / `Month` / `MonthName`。

点 **Save**，命名为 `WA Housing`。

---

## 第三步 · 建 DAX 度量

```
左侧栏 → 你的语义模型 → Open data model → 右键表名 → New measure
```

逐条粘贴。**每条粘完按回车保存再建下一条。**

### 核心口径 —— 这是整个作品的重点

```dax
Homes Sold = SUM('WA Housing'[HomesSold])
```

```dax
Active Inventory = SUM('WA Housing'[Inventory])
```

```dax
-- 不加权：39 个县各算一票，不管卖了 2 套还是 2000 套
Months of Supply (unweighted) = AVERAGE('WA Housing'[MonthsOfSupply])
```

```dax
-- 加权：把全州存量除以全州成交，等价于按成交量加权
Months of Supply (weighted) =
DIVIDE(
    SUM('WA Housing'[Inventory]),
    SUM('WA Housing'[HomesSold])
)
```

```dax
-- 两者之差，就是小县把数字推高了多少
Weighting Gap =
[Months of Supply (unweighted)] - [Months of Supply (weighted)]
```

### 其余度量

```dax
Sold Above List (unweighted) = AVERAGE('WA Housing'[SoldAboveListShare])
```

```dax
Sold Above List (weighted) =
DIVIDE(
    SUMX('WA Housing', 'WA Housing'[SoldAboveListShare] * 'WA Housing'[HomesSold]),
    SUM('WA Housing'[HomesSold])
)
```

```dax
Median Days on Market = AVERAGE('WA Housing'[MedianDOM])
```

```dax
Price Drop Share = AVERAGE('WA Housing'[PriceDropShare])
```

```dax
Median Sale Price = AVERAGE('WA Housing'[MedianSalePrice])
```

```dax
-- 只统计这个月成交量够大、指标才有意义的县
Counties Reporting =
CALCULATE(
    DISTINCTCOUNT('WA Housing'[County]),
    'WA Housing'[HomesSold] >= 20
)
```

```dax
-- 成交量低于 20 套的县，占了多少全州成交
Small County Share of Sales =
DIVIDE(
    CALCULATE(SUM('WA Housing'[HomesSold]), 'WA Housing'[HomesSold] < 20),
    SUM('WA Housing'[HomesSold])
)
```

### 同比（YoY）

```dax
Homes Sold YoY =
VAR Curr = [Homes Sold]
VAR Prior =
    CALCULATE(
        [Homes Sold],
        FILTER(
            ALL('WA Housing'),
            'WA Housing'[Year] = MAX('WA Housing'[Year]) - 1
                && 'WA Housing'[Month] = MAX('WA Housing'[Month])
        )
    )
RETURN DIVIDE(Curr - Prior, Prior)
```

```dax
Inventory YoY =
VAR Curr = [Active Inventory]
VAR Prior =
    CALCULATE(
        [Active Inventory],
        FILTER(
            ALL('WA Housing'),
            'WA Housing'[Year] = MAX('WA Housing'[Year]) - 1
                && 'WA Housing'[Month] = MAX('WA Housing'[Month])
        )
    )
RETURN DIVIDE(Curr - Prior, Prior)
```

### 市场判定 —— 让两个口径的分歧直接显示出来

```dax
Market Verdict (unweighted) =
VAR M = [Months of Supply (unweighted)]
RETURN
    SWITCH(
        TRUE(),
        M < 4, "Seller's market",
        M <= 6, "Balanced",
        "Buyer's market"
    )
```

```dax
Market Verdict (weighted) =
VAR M = [Months of Supply (weighted)]
RETURN
    SWITCH(
        TRUE(),
        M < 4, "Seller's market",
        M <= 6, "Balanced",
        "Buyer's market"
    )
```

⭐ **这两条会在同一个月给出相反的答案。那就是这个作品的论点。**

---

## 第四步 · 第 1 页：`How You Average Changes the Answer`

```
模型页面右上 → New report
```

### 顶部：四张卡片

| 卡片 | 度量 | 格式 |
|---|---|---|
| Homes Sold | `Homes Sold` | 千分位 |
| Active Inventory | `Active Inventory` | 千分位 |
| **Months of Supply (unweighted)** | `Months of Supply (unweighted)` | 1 位小数 |
| **Months of Supply (weighted)** | `Months of Supply (weighted)` | 1 位小数 |

🔴 **后两张卡片并排放，中间不要插别的。** 让 6.6 和 3.1 挨着。

再放两张 **Card**：`Market Verdict (unweighted)` 和 `Market Verdict (weighted)`
→ 会显示 "Buyer's market" 和 "Seller's market"

### 中部：折线图 —— 两条口径的走势

```
视觉对象：Line chart
X 轴：Date
Y 轴：Months of Supply (unweighted)  和  Months of Supply (weighted)
```
两条线从 2019 一路分开，**2026 差距最大**。

标题写：
> Same metric, two definitions — the gap widens as the market slows

### 底部：散点图 —— 谁在推高不加权口径

```
视觉对象：Scatter chart
X 轴：HomesSold（改成 Don't summarize）
Y 轴：MonthsOfSupply（改成 Average）
图例：County
筛选：Date is 2026-05-01
```
左上角那几个点（成交极少、可售月数极高）就是元凶。

**页面结论句（放标题下，字号 11-12）：**
> Nine counties account for 0.6% of Washington's May 2026 home sales but 23% of the weight in a 39-county average. Unweighted, the state reads as a buyer's market at 6.6 months of supply. Weighted by actual transactions, it reads as a seller's market at 3.1.

---

## 第五步 · 第 2 页：`Price Cuts and Time to Sell`

### 顶部：三张卡片

`Median Days on Market` · `Price Drop Share`（百分比）· `Counties Reporting`

### 主图：散点图

```
视觉对象：Scatter chart
X 轴：PriceDropShare（Average）
Y 轴：MedianDOM（Average）
图例：Year
大小：HomesSold（Sum）
筛选器 1：HomesSold >= 20
筛选器 2：Year >= 2024
```

🔴 **加趋势线：** 选中图 → 右侧 **Analytics**（放大镜图标）→ **Trend line** → 打开

### 副图：条形图 —— 证明不是季节性

```
视觉对象：Clustered column chart
X 轴：MonthName
Y 轴：MedianDOM（Average）
```
筛选 `PriceDropShare` 高低两组对比，或直接用下面这条度量：

```dax
-- 每个月内部，高降价县 vs 低降价县 的在售天数差
DOM Gap Within Month =
VAR HighDrop =
    CALCULATE(AVERAGE('WA Housing'[MedianDOM]),
        'WA Housing'[PriceDropShare] >= 0.3, 'WA Housing'[HomesSold] >= 20)
VAR LowDrop =
    CALCULATE(AVERAGE('WA Housing'[MedianDOM]),
        'WA Housing'[PriceDropShare] < 0.3, 'WA Housing'[HomesSold] >= 20)
RETURN HighDrop - LowDrop
```
放进条形图（X = MonthName，Y = `DOM Gap Within Month`）→ **12 根柱子应该全部为负。**

### 底部：分档表

```
视觉对象：Table 或 Clustered bar chart
```
需要先建一个分档列（在 data model 里 → New column）：

```dax
Price Drop Band =
SWITCH(
    TRUE(),
    'WA Housing'[PriceDropShare] < 0.2, "10-20%",
    'WA Housing'[PriceDropShare] < 0.3, "20-30%",
    'WA Housing'[PriceDropShare] < 0.4, "30-40%",
    "40-50%"
)
```

X = `Price Drop Band`，Y = `Median Days on Market`，筛选 `HomesSold >= 20` 且 `Year >= 2024`
→ 应该看到 **63 → 40 → 27 → 23**

**页面结论句：**
> Counties with more price cuts sell faster, not slower — median days on market falls from 63 to 23 as the price-cut share rises. The relationship holds inside all twelve calendar months, so it is not a summer effect. This is an association, not a causal estimate.

---

## 第六步 · 保存并导出

1. **Save** → 命名 `Washington Housing Market — Definition Sensitivity`
2. **截图两页**（Mac：`Cmd + Shift + 4`）
   → 存成 `images/page1-averaging.png` 和 `images/page2-price-cuts.png`
3. **下载 .pbix**：报表页面 → `File` → `Download this file` → 选 **A copy of your report and data**
   - ✅ 因为数据源是 Web URL 不是上传的 CSV，这里应该能下载
   - ❌ 如果灰掉：跳过，用截图 + DAX 代码就够了（README 里已写明）

4. 放进仓库：
```bash
cd ~/Desktop/wa-housing-powerbi
mkdir -p images
# 把截图和 .pbix 拷进来，然后：
git add -A && git commit -m "Add Power BI report, screenshots and measures" && git push
```

---

## 🔴 做完之后，简历怎么写

**可以写：**
> Built a two-page Power BI report on Redfin's public county-level data (39 Washington counties, 2019–2026), modeled in Power Query with DAX measures.

**核心那句：**
> Found that months of supply — the standard measure of market balance — gives opposite verdicts depending on whether counties are weighted by transaction volume: 6.6 months (buyer's market) unweighted versus 3.1 (seller's market) weighted, because nine counties representing 0.6% of sales carry 23% of the weight in a simple county average.

**🔴 仍然不可以写：** `expert in Power BI` · `production Power BI deployment` · 任何暗示这是工作产出的说法
**✅ 可以写：** `Power BI (personal project)` 或放在 Analytics Portfolio 栏

---

## 如果网页版不够用

**备选 A —— 借一台 Windows 电脑：** Power BI Desktop 从 Microsoft Store 免费装，3 小时做完。

**备选 B —— 腾磁盘走虚拟机：** 需要再腾出约 15 GB（现在 27.6 GB，需要 40 GB）。
```
Apple 菜单 → 系统设置 → 通用 → 储存空间
```
优先清 `开发者`（Xcode 缓存）和 `废纸篓`。
