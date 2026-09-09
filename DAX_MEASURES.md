# DAX 全清单 — WA Housing Market

**表名假设是 `wa_housing_county_month`**（Power BI 通常按 CSV 文件名命名）。
如果你的表叫别的名字，把下面所有 `'wa_housing_county_month'` 替换掉即可。

**建法：** Data 面板 → 表名右键（或 `...`）→ `New measure` → 粘贴 → 回车 → 建下一条。

---

## A · 第 1 页必需（7 条）

### 基础三条

```dax
Homes Sold = SUM('wa_housing_county_month'[HomesSold])
```

```dax
Active Inventory = SUM('wa_housing_county_month'[Inventory])
```

```dax
Median Sale Price = AVERAGE('wa_housing_county_month'[MedianSalePrice])
```

### ⭐ 核心两条 —— 整个作品的论点

```dax
Months of Supply (unweighted) = AVERAGE('wa_housing_county_month'[MonthsOfSupply])
```
> 39 个县各算一票。Lincoln 卖 2 套和 King 卖 2,134 套同权。

```dax
Months of Supply (weighted) =
DIVIDE(
    SUM('wa_housing_county_month'[Inventory]),
    SUM('wa_housing_county_month'[HomesSold])
)
```
> 全州存量 ÷ 全州成交，等价于按成交量加权。

```dax
Weighting Gap = [Months of Supply (unweighted)] - [Months of Supply (weighted)]
```

### 判定两条 —— 直接显示成文字

```dax
Market Verdict (unweighted) =
VAR M = [Months of Supply (unweighted)]
RETURN
    SWITCH(
        TRUE(),
        M < 4,  "Seller's market",
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
        M < 4,  "Seller's market",
        M <= 6, "Balanced",
        "Buyer's market"
    )
```

⭐ **这两条在同一个月给出相反答案。放两张 Card 并排，就是整页的论点。**

---

## B · 支撑「谁在推高不加权口径」（3 条）

```dax
Small County Share of Sales =
DIVIDE(
    CALCULATE(
        SUM('wa_housing_county_month'[HomesSold]),
        'wa_housing_county_month'[HomesSold] < 20
    ),
    SUM('wa_housing_county_month'[HomesSold])
)
```
> 月成交 <20 套的县，占全州成交多少。2026-05 = **0.62%**

```dax
Counties Reporting =
CALCULATE(
    DISTINCTCOUNT('wa_housing_county_month'[County]),
    'wa_housing_county_month'[HomesSold] >= 20
)
```
> 2026-05 = **30**（全州 39 个县）

```dax
Small County Weight in Average =
DIVIDE(
    CALCULATE(
        DISTINCTCOUNT('wa_housing_county_month'[County]),
        'wa_housing_county_month'[HomesSold] < 20
    ),
    DISTINCTCOUNT('wa_housing_county_month'[County])
)
```
> 那 9 个县在 39 县均值里占的权重 = **23%**。
> **0.62% 的成交 vs 23% 的权重 —— 这就是全部原因。**

---

## C · 第 2 页必需（4 条 + 1 个计算列）

```dax
Median Days on Market = AVERAGE('wa_housing_county_month'[MedianDOM])
```

```dax
Price Drop Share = AVERAGE('wa_housing_county_month'[PriceDropShare])
```

```dax
Sold Above List (weighted) =
DIVIDE(
    SUMX(
        'wa_housing_county_month',
        'wa_housing_county_month'[SoldAboveListShare] * 'wa_housing_county_month'[HomesSold]
    ),
    SUM('wa_housing_county_month'[HomesSold])
)
```
> 2026-05 = **30.9%**（不加权是 22.6% —— 同一个偏差）

### 🔴 计算列（不是度量）

**建法不一样：** 表名右键 → **`New column`**

```dax
Price Drop Band =
SWITCH(
    TRUE(),
    ISBLANK('wa_housing_county_month'[PriceDropShare]), "(not reported)",
    'wa_housing_county_month'[PriceDropShare] < 0.10, "<10%",
    'wa_housing_county_month'[PriceDropShare] < 0.20, "10-20%",
    'wa_housing_county_month'[PriceDropShare] < 0.30, "20-30%",
    'wa_housing_county_month'[PriceDropShare] < 0.40, "30-40%",
    "40%+"
)
```

### ⭐ 证明不是季节性的那条

```dax
DOM Gap Within Month =
VAR HighDrop =
    CALCULATE(
        AVERAGE('wa_housing_county_month'[MedianDOM]),
        'wa_housing_county_month'[PriceDropShare] >= 0.3,
        'wa_housing_county_month'[HomesSold] >= 20
    )
VAR LowDrop =
    CALCULATE(
        AVERAGE('wa_housing_county_month'[MedianDOM]),
        'wa_housing_county_month'[PriceDropShare] < 0.3,
        'wa_housing_county_month'[HomesSold] >= 20
    )
RETURN HighDrop - LowDrop
```
> 放进条形图（X = `MonthName`，Y = 这条）→ **12 根柱子应该全部为负**，
> 说明「降价的卖得快」在每个日历月内部都成立，不是夏天效应。

---

## D · 同比（可选，2 条）

```dax
Homes Sold YoY =
VAR Curr = [Homes Sold]
VAR Prior =
    CALCULATE(
        [Homes Sold],
        FILTER(
            ALL('wa_housing_county_month'),
            'wa_housing_county_month'[Year] = MAX('wa_housing_county_month'[Year]) - 1
                && 'wa_housing_county_month'[Month] = MAX('wa_housing_county_month'[Month])
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
            ALL('wa_housing_county_month'),
            'wa_housing_county_month'[Year] = MAX('wa_housing_county_month'[Year]) - 1
                && 'wa_housing_county_month'[Month] = MAX('wa_housing_county_month'[Month])
        )
    )
RETURN DIVIDE(Curr - Prior, Prior)
```

---

## 🔴 建完必须核对（筛选到 2026-05）

**这些是我在 Python 里独立算出来的真值。DAX 算不出这些数就是写错了。**

| 度量 | 应该等于 |
|---|---|
| Homes Sold | **7,884** |
| Active Inventory | **24,287** |
| Months of Supply (unweighted) | **6.62** |
| Months of Supply (weighted) | **3.08** |
| Weighting Gap | **3.54** |
| Market Verdict (unweighted) | **Buyer's market** |
| Market Verdict (weighted) | **Seller's market** |
| Small County Share of Sales | **0.62%** |
| Counties Reporting | **30** |
| Small County Weight in Average | **23%** |
| Price Drop Share | **29.8%** |
| Sold Above List (weighted) | **30.9%** |

**散点样本（2024 起 · 月成交 ≥20 · n=834）分档中位在售天数：**

| Price Drop Band | Median DOM |
|---|---|
| 10-20% | **63** |
| 20-30% | **40** |
| 30-40% | **27** |
| 40%+ | **24** |

---

## 优先级

**只想先出第 1 页截图 → 建 A 组 7 条就够。**
**要完整两页 → A + B + C 共 14 条 + 1 个计算列。**
D 组同比是锦上添花，可以不做。

---

## ⚠️ 如果 `New measure` 找不到

那就在 **Power Query** 里用 `Group by` + `Custom Column` 做等价的聚合（M 语言）。
结果一样，但那是 Power Query 不是 DAX ——
🔴 **简历上就只能写 `Power Query modeling`，要把 `DAX measures` 去掉。**
