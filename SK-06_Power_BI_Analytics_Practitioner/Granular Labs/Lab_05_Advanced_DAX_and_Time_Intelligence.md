# Lab 05 — Advanced DAX & Time Intelligence

> **Goal:** Complete the guided lab's measure library: fiscal YTD/QTD, prior year and growth, a correct 3-month moving average, rolling 12 months, revenue targets through an inactive relationship, a RAG status, ship-date revenue, new vs repeat customers and the 80/20 rule — each one checked against exact expected numbers.
> **Time:** 5–6 hours | **Difficulty:** Intermediate → Advanced | **Prerequisites:** Lab 04 (`NorthStar_Lab04.pbix` with the `Sales` and `Analysis` measures)
> **Guided lab coverage:** PART 3 Steps 3.2 (time intelligence), 3.3 (KPI) and 3.4 (advanced measures)
> **Produces:** `C:\DataEngineering\sk06\pbix\NorthStar_Lab05.pbix` — the full 27-measure library

## Learning objectives

By the end of this lab you can:

- Explain time intelligence as `CALCULATE` + a date filter, and its prerequisites (marked, contiguous date table; date fields from `dim_date`).
- Write fiscal YTD with `DATESYTD(…, "6/30")` and say why QTD needs no fiscal parameter for NorthStar.
- Build prior-year and growth measures that return BLANK — not 0 — when there is nothing to compare.
- Explain why averaging over the *dates* of a 3-month window gives a daily, not monthly, average.
- Activate an inactive relationship with `USERELATIONSHIP` (targets by month, revenue by ship date).
- Write the RAG status, new-vs-repeat and Pareto measures, and reconcile them with SQL.

**Key terms**

| Term | Meaning |
|---|---|
| **Time intelligence** | DAX functions that return a set of dates (`DATESYTD`, `SAMEPERIODLASTYEAR`, `DATESINPERIOD`) for `CALCULATE` to use as a filter. |
| **Fiscal year end** | The `"6/30"` argument of `DATESYTD`: NorthStar's year ends 30 June, so YTD restarts on 1 July. |
| **PY (prior year)** | The same dates shifted back one year (`SAMEPERIODLASTYEAR`). Calendar-agnostic: it shifts whatever dates are filtered. |
| **Rolling window** | `DATESINPERIOD(dates, end, -N, MONTH)` — the N months ending at a date; does not reset at year end. |
| **USERELATIONSHIP** | A `CALCULATE` modifier that switches on an inactive relationship for one measure only. |
| **RAG status** | Red / Amber / Green traffic light. Here 1 = green, 0 = amber, −1 = red, BLANK = no target. |
| **Pareto (80/20)** | A small share of items (products) produces most of the value (revenue). |

**How time intelligence works under the hood.** `DATESYTD(dim_date[Date], "6/30")` looks at the last date in the current filter context, and returns all dates from the previous 1 July up to it. `CALCULATE` swaps that set in as the date filter. Because `dim_date` is **marked as the date table** (Lab 03), filtering its `Date` column also removes the other `dim_date` filters (such as `Year-Month`) inside that `CALCULATE` — which is what lets YTD "see" earlier months. Use date fields from `dim_date` in visuals, never from a fact.

## Steps

### Step 1 — Open a copy and build the test matrix

1. `docker compose start` in `C:\DataEngineering\sk06`.
2. Open `NorthStar_Lab04.pbix` → **Save As** `C:\DataEngineering\sk06\pbix\NorthStar_Lab05.pbix`.
3. Preconditions (each one breaks this lab if missing): `dim_date` **marked as date table**; `Year-Month` sorted by `Year-Month Sort`; Auto date/time **off**; `Fiscal Year` set to *Don't summarize* (all from Labs 00 and 03).
4. New page **TI Test**: a **Slicer** on `dim_date[Fiscal Year]` = **2026**, and a **Matrix** with rows `dim_date[Year-Month]` and values `Net Revenue`.

Every measure in Steps 2–6 goes into `_Measures`, display folder **`Time Intelligence`**, format Currency 2 decimals (percentages: Percentage 2 decimals), and gets added to this matrix as you go. The full expected matrix is at the end of Step 6.

✅ **Checkpoint:** the matrix lists 2025-07 … 2026-06 in that order; 2025-07 shows $1,032,585.71.

### Step 2 — Fiscal year-to-date and quarter-to-date

```dax
// ── WHY the "6/30" fiscal year end parameter ───────────────────
// NorthStar's fiscal year ends June 30. Without this parameter,
// DATESYTD starts accumulating from January 1. With it, the YTD
// resets at the correct fiscal year boundary.

Revenue YTD =
CALCULATE(
    [Net Revenue],
    DATESYTD(dim_date[Date], "6/30")
)
```

```dax
// Fiscal quarters (Jul–Sep, Oct–Dec, ...) line up with calendar
// quarters, so DATESQTD needs no fiscal parameter.
Revenue QTD =
CALCULATE(
    [Net Revenue],
    DATESQTD(dim_date[Date])
)
```

**Verify (the smoking-gun test):**

- **2025-07:** `Revenue YTD` = `Net Revenue` = **$1,032,585.71** — the year just restarted. If July shows a big accumulated number, the `"6/30"` is missing.
- **2025-08:** YTD = July + August = **$2,043,064.95**.
- **2025-12:** YTD = **$6,475,316.27**; `Revenue QTD` = Oct + Nov + Dec = **$3,566,946.98**.
- **2026-06:** YTD = **$11,862,350.26** — the whole fiscal year, equal to the control total.

Temporarily remove `"6/30"` from the formula and look at 2026-01: YTD drops to January alone ($868,589.28) — calendar YTD, wrong for NorthStar. Put it back.

✅ **Checkpoint:** July equals Net Revenue; June equals $11,862,350.26.

### Step 3 — Prior year and growth

```dax
// Prior year same period — the CFO's most-requested benchmark
Revenue PY =
CALCULATE(
    [Net Revenue],
    SAMEPERIODLASTYEAR(dim_date[Date])
)
```

```dax
// WHY BLANK() not 0: If there is no prior year data (first year),
// showing 0% growth is misleading. BLANK() correctly shows no comparison.
YoY Growth % =
VAR CurrentRevenue   = [Net Revenue]
VAR PriorYearRevenue = [Revenue PY]
RETURN
DIVIDE(
    CurrentRevenue - PriorYearRevenue,
    PriorYearRevenue,
    BLANK()
)
```

```dax
// YTD of the prior year (for YTD growth tracking): shift the dates back
// a year, then let [Revenue YTD] apply the fiscal YTD logic
Revenue PY YTD =
CALCULATE(
    [Revenue YTD],
    SAMEPERIODLASTYEAR(dim_date[Date])
)
```

```dax
YTD Growth % =
DIVIDE(
    [Revenue YTD] - [Revenue PY YTD],
    [Revenue PY YTD],
    BLANK()
)
```

**How to read them:** `SAMEPERIODLASTYEAR` shifts the *currently filtered* dates back one year. Fiscal awareness comes from `dim_date` (the rows you grouped by) and `DATESYTD`; shifts need no fiscal parameter. `Revenue PY YTD` composes two verified measures instead of nesting time functions — easier to read and to test.

**Verify:**

- 2025-07: `Revenue PY` = **$881,559.21** (July 2024), `YoY Growth %` = **17.13%**.
- 2025-12: `Revenue PY YTD` = **$5,866,831.07**, `YTD Growth %` = **10.37%**.
- Matrix total (FY2026): `YoY Growth %` = **10.01%**.
- Switch the slicer to **2024**: `Revenue PY` and `YoY Growth %` are **blank** — there is no FY2023 data. Blank is correct, not a bug. Switch back to 2026.

✅ **Checkpoint:** FY2026 YoY Growth % = 10.01%; FY2024 shows blanks, not 0% or errors.

### Step 4 — A correct 3-month moving average and rolling 12 months

The guided lab's original moving average was:

```dax
// DON'T USE — kept to show the bug
Revenue 3M Moving Avg (daily) =
AVERAGEX(
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -3, MONTH),
    CALCULATE([Net Revenue])
)
```

`DATESINPERIOD` returns about **92 dates**, so `AVERAGEX` averages *daily* revenues: a figure of roughly $40,000 next to monthly revenues of about $1,000,000 — the dotted line would hug the bottom of the chart. Create it, look at 2025-12 (tens of thousands, not a million), then **delete** it and create the correct version:

```dax
// 3-month moving average smooths holiday/promotion spikes.
// Total revenue of the last 3 months divided by 3 = average MONTHLY revenue.
// (Averaging over the dates of the window would give average DAILY revenue.)
Revenue 3M Moving Avg =
VAR Last3Months =
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -3, MONTH)
RETURN
DIVIDE(CALCULATE([Net Revenue], Last3Months), 3)
```

```dax
// Rolling 12 months (does not reset at year end — unlike YTD)
Revenue R12M =
CALCULATE(
    [Net Revenue],
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -12, MONTH)
)
```

`MAX(dim_date[Date])` is the last date of the cell (31 Dec for 2025-12); the window reaches back 3 (or 12) months from it — across the fiscal-year boundary when needed, because filtering the marked date column replaces the slicer's year filter inside `CALCULATE`.

**Verify:** 2025-12: `Revenue 3M Moving Avg` = (998,699.78 + 1,218,920.12 + 1,349,327.08) / 3 = **$1,188,982.33**. 2025-07 = **$938,066.03** (May, June and July 2025 — the window crosses into FY2025). 2026-06: `Revenue R12M` = **$11,862,350.26**.

Limitation to state out loud: for the first two months of data (2023-07, 2023-08) the window has fewer than three months of sales, so the average is understated.

✅ **Checkpoint:** 2025-12 moving average $1,188,982.33; the daily-average version is deleted.

### Step 5 — Targets through the inactive relationship (display folder `KPI`)

First see the problem. Create a quick test measure:

```dax
Target (no USERELATIONSHIP) =
SUM(fact_targets[revenue_target])
```

Add it to the matrix: every month shows **$34,060,900.00** — all targets of all years. The relationship `fact_targets[target_date_key]` → `dim_date[DateKey]` is inactive (Lab 03 Step 4), so no date filter reaches the targets. (This is guided lab Reflection Question 1.) Delete the test measure and create the real ones:

```dax
// Revenue Target — from fact_targets
// WHY USERELATIONSHIP: fact_targets[target_date_key] is linked to
// dim_date via an INACTIVE relationship (Step 2.4). USERELATIONSHIP
// activates it within this measure only, so the year/month slicers that
// filter fact_sales also filter fact_targets.
Revenue Target =
CALCULATE(
    SUM(fact_targets[revenue_target]),
    USERELATIONSHIP(fact_targets[target_date_key], dim_date[DateKey])
)
```

```dax
Revenue Variance =
[Net Revenue] - [Revenue Target]
```

```dax
Revenue Variance % =
DIVIDE([Revenue Variance], [Revenue Target], BLANK())
```

```dax
// Traffic light for conditional formatting on KPI cards
// Returns 1 (green), 0 (yellow), -1 (red) — or BLANK when there is no target
Revenue Status =
VAR VarPct = [Revenue Variance %]
RETURN
IF(
    ISBLANK(VarPct),
    BLANK(),                // no target in this context: no colour, not "green"
    SWITCH(
        TRUE(),
        VarPct >= 0,     1,     // Green: at or above target
        VarPct >= -0.10, 0,     // Yellow: within 10% below target
        -1                      // Red: more than 10% below target
    )
)
```

**Why the `ISBLANK` guard?** Without it, a context with no target (for example FY2027) gives a blank variance, and `BLANK() >= 0` is **TRUE** in DAX — the status would show *green* for "no data". The original guided lab had exactly this bug.

`SWITCH(TRUE(), condition1, result1, …, else)` returns the result of the first true condition — a readable "if / else if" chain.

**Verify** — new matrix: rows `dim_store[region_name]`, values `Net Revenue`, `Revenue Target`, `Revenue Variance %`, `Revenue Status`; slicer FY2026:

| region_name | Net Revenue | Revenue Target | Revenue Variance % | Revenue Status |
|---|---|---|---|---|
| Midwest | $3,053,466.54 | $3,263,200.00 | -6.43% | 0 |
| Northeast | $3,346,490.01 | $3,239,600.00 | 3.30% | 1 |
| Southeast | $3,103,316.68 | $3,191,600.00 | -2.77% | 0 |
| West | $2,359,077.03 | $2,688,900.00 | -12.27% | -1 |
| **Total** | **$11,862,350.26** | **$12,383,300.00** | **-4.21%** | **0** |

Control query in psql:

```sql
SELECT r.region_name, SUM(t.revenue_target) AS revenue_target
FROM finance.revenue_targets t
JOIN sales.stores s    ON s.store_id = t.store_id
JOIN sales.districts d ON d.district_id = s.district_id
JOIN sales.regions r   ON r.region_id = d.region_id
WHERE t.target_month BETWEEN DATE '2025-07-01' AND DATE '2026-06-01'
GROUP BY r.region_name
ORDER BY r.region_name;
```

**Expected output:** Midwest 3263200.00, Northeast 3239600.00, Southeast 3191600.00, West 2688900.00.

✅ **Checkpoint:** targets now follow the Fiscal Year slicer and the four statuses are 0, 1, 0, −1.

### Step 6 — Shipping and advanced measures

**Display folder `Shipping`:**

```dax
// ── Ship Date Revenue (inactive relationship activation) ────────
// The logistics team reports by ship date, not order date.
// USERELATIONSHIP activates the inactive ShipDateKey → dim_date
// relationship for this measure only.
Revenue by Ship Date =
CALCULATE(
    [Net Revenue],
    USERELATIONSHIP(fact_sales[ShipDateKey], dim_date[DateKey])
)
```

```dax
// Average days from order to shipment, per order line.
// days_to_ship was computed in Power Query (Step 2.2); unshipped
// lines have a null value, and AVERAGE ignores blanks.
Avg Days to Ship =
AVERAGE(fact_sales[days_to_ship])
```

The original guided lab rebuilt dates from the integer keys inside `AVERAGEX`. That crashes into the never-shipped lines (blank `ShipDateKey` → `DATE(0, 0, 0)`, a date in 1899) and repeats work on every query. Computing `days_to_ship` once per row in Power Query is simpler, faster and correct.

**Display folder `Advanced`:**

```dax
// "New" = the customer registered inside the selected period
New Customer Revenue =
CALCULATE(
    [Net Revenue],
    FILTER(
        dim_customer,
        dim_customer[registration_date] >= MIN(dim_date[Date]) &&
        dim_customer[registration_date] <= MAX(dim_date[Date])
    )
)
```

```dax
Repeat Customer Revenue =
[Net Revenue] - [New Customer Revenue]
```

```dax
Repeat Customer % =
DIVIDE([Repeat Customer Revenue], [Net Revenue], BLANK())
```

```dax
// ── Pareto: share of revenue from the top 20% of products ──────
Top 20% Products Revenue % =
VAR ProductsInScope = VALUES(dim_product[product_id])
VAR TopCount        = ROUNDUP(COUNTROWS(ProductsInScope) * 0.2, 0)
VAR TopProducts     = TOPN(TopCount, ProductsInScope, [Net Revenue], DESC)
RETURN
DIVIDE(CALCULATE([Net Revenue], TopProducts), [Net Revenue], BLANK())
```

**Reading the advanced measures:**

- `FILTER(dim_customer, …)` is an iterator over 30,000 customers; inside it, `MIN(dim_date[Date])` / `MAX(dim_date[Date])` still see the slicer's period (row context on customers does not filter dates). The surviving customers become a filter for `CALCULATE`.
- `TOPN(2400, products, [Net Revenue])` returns the 2,400 best-selling products in the current context (20% of 12,000); `CALCULATE` then sums revenue for just those products.

**Verify** — cards with FY2026 slicer:

| Measure | Expected (FY2026) |
|---|---|
| Revenue by Ship Date | $11,709,129.07 |
| Avg Days to Ship | 3.42 (format: decimal, 2 places) |
| New Customer Revenue | $2,236,771.55 |
| Repeat Customer Revenue | $9,625,578.71 |
| Repeat Customer % | 81.14% |
| Top 20% Products Revenue % | 80.07% |

`Revenue by Ship Date` is lower than `Net Revenue` because it counts lines by the date they **shipped**: late-June-2025 orders shipped in July count in FY2026, but late-June-2026 orders shipping in July 2026, and never-shipped orders, do not. Control query:

```sql
SELECT ROUND(SUM(ol.quantity * ol.unit_price - ol.discount_amount), 2) AS revenue_by_ship_date
FROM sales.orders o
JOIN sales.order_lines ol ON ol.order_id = o.order_id
WHERE o.ship_date BETWEEN DATE '2025-07-01' AND DATE '2026-06-30';
```

**Expected output:** `11709129.07`.

`Top 20% Products Revenue % = 80.07%`: NorthStar's catalogue follows the 80/20 rule almost exactly — a merchandising insight the CFO will want on the Product page.

✅ **Checkpoint:** all six values match.

### Step 7 — The full expected test matrix

Your **TI Test** matrix (FY2026, rows `Year-Month`) should now read:

| Year-Month | Net Revenue | Revenue YTD | Revenue PY | YoY Growth % | Revenue 3M Moving Avg | Revenue Target | Revenue Variance % |
|---|---|---|---|---|---|---|---|
| 2025-07 | $1,032,585.71 | $1,032,585.71 | $881,559.21 | 17.13% | $938,066.03 | $1,083,600.00 | -4.71% |
| 2025-08 | $1,010,479.24 | $2,043,064.95 | $907,984.24 | 11.29% | $971,975.39 | $1,057,000.00 | -4.40% |
| 2025-09 | $865,304.34 | $2,908,369.29 | $830,597.28 | 4.18% | $969,456.43 | $904,600.00 | -4.34% |
| 2025-10 | $998,699.78 | $3,907,069.07 | $859,727.97 | 16.16% | $958,161.12 | $1,037,100.00 | -3.70% |
| 2025-11 | $1,218,920.12 | $5,125,989.19 | $1,095,230.52 | 11.29% | $1,027,641.41 | $1,271,500.00 | -4.14% |
| 2025-12 | $1,349,327.08 | $6,475,316.27 | $1,291,731.85 | 4.46% | $1,188,982.33 | $1,408,100.00 | -4.17% |
| 2026-01 | $868,589.28 | $7,343,905.55 | $756,173.03 | 14.87% | $1,145,612.16 | $906,700.00 | -4.20% |
| 2026-02 | $726,884.53 | $8,070,790.08 | $710,579.86 | 2.29% | $981,600.30 | $761,700.00 | -4.57% |
| 2026-03 | $938,565.04 | $9,009,355.12 | $846,096.75 | 10.93% | $844,679.62 | $977,400.00 | -3.97% |
| 2026-04 | $891,396.41 | $9,900,751.53 | $821,740.64 | 8.48% | $852,281.99 | $934,400.00 | -4.60% |
| 2026-05 | $1,019,186.82 | $10,919,938.35 | $908,751.16 | 12.15% | $949,716.09 | $1,058,700.00 | -3.73% |
| 2026-06 | $942,411.91 | $11,862,350.26 | $872,861.21 | 7.97% | $950,998.38 | $982,500.00 | -4.08% |

(At the **Total** row the time-intelligence measures evaluate for the last date of the year, so `Revenue YTD` total = $11,862,350.26 and `Revenue 3M Moving Avg` total = the June value.)

✅ **Checkpoint:** every cell matches. If one column is off, the table above tells you which measure to fix.

### Step 8 — Organise, document, regression-check, save

1. Display folders: `Sales` (8), `Analysis` (1), `Time Intelligence` (8), `KPI` (4), `Shipping` (2), `Advanced` (4) — **27 measures** in total, the guided lab's library.
2. A one-sentence **Description** for every new measure.
3. Regression check: `Net Revenue` for FY2026 is still **$11,862,350.26** — adding derived measures must never move base numbers.
4. **Ctrl+S**.

✅ **Checkpoint:** six display folders; no measure outside a folder; the control total is unchanged.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| YTD equals Net Revenue in every month | Date field in the visual not from `dim_date`, or `dim_date` not marked | Use `dim_date[Year-Month]`; Table tools → Mark as date table. |
| YTD restarts in January | `"6/30"` missing | `DATESYTD(dim_date[Date], "6/30")`. |
| YTD blank everywhere | Relationship `fact_sales[OrderDateKey]` → `dim_date[DateKey]` missing or inactive | Lab 03 Step 2. |
| Moving average around $40,000 | Averaging over days (the original bug) | Use the `DIVIDE(CALCULATE(…, Last3Months), 3)` version. |
| Revenue Target is the same in every month | `USERELATIONSHIP` missing, or relationship not created | Check the measure; Model view must show the dashed `target_date_key` line. |
| *USERELATIONSHIP function can only use the two columns references participating in relationship* | No relationship exists between those columns | Create it (inactive) in Lab 03 Step 4. |
| Revenue Status shows 1 for FY2027 | `ISBLANK` guard missing | Use the guarded version above. |
| New Customer Revenue equals Net Revenue | `registration_date` typed as text | Power Query: type `registration_date` as Date; refresh. |
| Top 20% measure is slow in a product-level table | `TOPN` re-evaluated for every row | Use it on cards or at category level, as intended. |

## Key takeaways

- Time intelligence = `CALCULATE` + a set of dates. Prerequisites — marked, contiguous `dim_date` and date fields taken from it — are the mechanism, not bureaucracy.
- One fiscal definition in one place: `dim_date` columns plus `"6/30"` in `DATESYTD`. Shifts (`SAMEPERIODLASTYEAR`) are calendar-agnostic.
- Guard the edges: BLANK means "no comparison", 0 means a value. `BLANK() >= 0` is TRUE — guard RAG logic with `ISBLANK`.
- `USERELATIONSHIP` activates an inactive relationship for one measure: targets by month, revenue by ship date.
- FY2026: YoY +10.01%, YTD growth at December +10.37%, revenue 4.21% below target (Northeast the only region above), 81.14% from repeat customers, 80.07% from the top 20% of products.

**Interview check:** *Your company's fiscal year starts in July — what changes in your DAX?* — `DATESYTD`/`TOTALYTD` get the year-end argument `"6/30"`; the date table carries fiscal year/quarter/month columns; shifts like `SAMEPERIODLASTYEAR` do not change. *A target measure shows the full-year total in every month — why?* — The date relationship to the target table is inactive or missing, and the measure does not activate it with `USERELATIONSHIP`.

## Next

[Lab 06 — Report Design, Dashboards, RLS, Performance & Publishing](Lab_06_Report_Design_Dashboards_RLS_Performance_Publishing.md): build the Executive Summary page, add row-level security for regional and store managers, tune performance and publish.
