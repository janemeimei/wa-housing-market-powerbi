# Washington Housing Market — When the Definition Decides the Answer

**39 counties · 89 months · 2019-01 to 2026-05 · Redfin public data · Power BI**

Months of supply is the standard measure of whether a housing market favours buyers or
sellers. Under four months is a seller's market; over six is a buyer's market.

For Washington State in May 2026, the same source data produces both answers.

| | Months of supply | Verdict |
|---|---|---|
| Average of 39 counties | **6.6** | **Buyer's market** |
| Weighted by actual sales | **3.1** | **Seller's market** |

Neither number is wrong. They answer different questions, and only one of them is
the question anyone actually means.

---

## 1. Why the two numbers disagree

Nine of Washington's 39 counties sold fewer than 20 homes in May 2026. Together they
account for **0.62% of the state's sales** — and **23% of the weight** in a simple
county average.

| County | Homes sold | Months of supply | Weight in a county average |
|---|---|---|---|
| **Lincoln** | **2** | **47.5** | 1/39 |
| **King** | **2,134** | **2.8** | 1/39 |

Lincoln County had 95 listings and sold two houses. That is a real ratio, and for
Lincoln County it is meaningful. Averaged into a state figure with every county
counting once, it moves the number that decides how the whole state gets described.

The nine smallest counties average **14.5 months of supply**. The thirty that carry
99.4% of sales average **4.2**.

**This is not a rounding issue. It is the difference between two published verdicts.**

---

## 2. The gap is not stable, so you cannot correct for it once

The unweighted figure is higher in **100% of the 89 months** in the data — but by an
amount that changes with the market.

| | Unweighted | Weighted | Gap |
|---|---|---|---|
| May 2019 | 3.3 | 1.9 | **1.3** |
| May 2021 | 1.5 | 0.8 | **0.8** |
| May 2026 | 6.6 | 3.1 | **3.5** |

As the market slows, thin counties slow disproportionately, and the two definitions
diverge fastest exactly when the reading matters most. A constant adjustment factor
would not have worked.

---

## 3. A second finding: price cuts track with faster sales, not slower

The intuitive reading of a price reduction is weakness — a seller who cannot move
the property. The data points the other way.

**Counties with a higher share of listings cutting price sell faster.**

| Share of listings with a price drop | Median days on market |
|---|---|
| 10–20% | **63** |
| 20–30% | **40** |
| 30–40% | **27** |
| 40–50% | **23** |

r = **−0.50** across 834 county-months (2024 onward, counties with 20+ monthly sales).

### Controlling for seasonality

Summer has both more price cuts and faster sales, so the raw correlation could be
calendar effects. Removing the month means:

**r = −0.37**, and the relationship is negative **inside all twelve calendar months**
(range −0.59 to −0.26).

About a quarter of the raw correlation was seasonal. The rest was not.

**Reading:** a price cut is not a symptom of a stalled market — it is the seller moving
toward the clearing price. The markets that stall are the ones where asking prices
do not move.

🔴 **This is an association, not a causal estimate.** Counties differ in price tier,
housing mix, and demand, and none of that is controlled for here.

---

## 4. What was done to the data before any of this

**Property type is not mutually exclusive.** Redfin's `PROPERTY_TYPE` has five values,
and `All Residential` is the sum of the other four. Keeping all five and summing would
double every state-level total. Filtered to `All Residential` only: **12,512 rows → 3,451**,
one row per county-month.

**Missing `PRICE_DROPS` is not random.** 11.6% of rows are missing it, and they are
concentrated in small counties — median 37 monthly sales versus 70 where it is present.
Those rows carry 3.95% of the state's transactions. The scatter analysis filters to
counties with 20+ monthly sales for a separate reason (the ratio is unstable below
that), which also removes most of the missingness.

**No rows were dropped for being extreme.** Lincoln County's 47.5 months of supply is
not an outlier to be cleaned — it is the observation the whole first section is about.

---

## 5. The report

![Page 1 — how you average changes the answer](images/page1-averaging.png)

![Page 2 — price cuts and time to sell](images/page2-price-cuts.png)

---

## 6. Files

```
data/wa_housing_county_month.csv     3,451 rows, one per county-month
DAX_MEASURES.md                      every measure, with the value each should return
BUILD_GUIDE.md                       how this was built
                                     in the browser without Power BI Desktop
images/                              report screenshots
```

Data modelled in **Power Query**, measures written in **DAX**, two report pages.
`BUILD_GUIDE.md` documents the full build — every measure, every visual's field
configuration, and the browser-only authoring path for anyone without a Windows
machine, including the `.pbix` download restriction that made a hosted data source
necessary rather than an uploaded file.

**Source:** [Redfin Data Center](https://www.redfin.com/news/data-center/),
county_market_tracker. Public data, free to use with attribution.

---

## 7. What this does not establish

- **County-level aggregates only.** Nothing here sees within-county variation, and
  price tier and housing mix are uncontrolled.
- **The price-cut relationship is an association.** A matched comparison of similar
  properties would be needed for a causal claim.
- **Redfin covers listings on the MLS.** Off-market and new-construction sales are
  under-represented, and that coverage plausibly differs by county.
- **Months of supply itself is a ratio of a stock to a flow.** Both definitions here
  inherit that; neither is a forecast.

---

**Jane (Jingjie) Mei** · Sammamish, WA · M.P.S. Analytics, Northeastern University
[GitHub](https://github.com/janemeimei) · [LinkedIn](https://linkedin.com/in/janemei-analytics) · [Tableau](https://public.tableau.com/app/profile/jane.mei)
