# Lab 04 — DAX Fundamentals: Measures, Filter Context & CALCULATE

> **Goal:** Build the guided lab's base measure library (Net Revenue, Total Cost, Gross Margin, Gross Margin %, Order Count, Average Order Value, Quantity Sold, Return Rate) plus `Revenue % of Company Total`, understand *why* each number changes per visual cell, and reconcile every measure with SQL control totals.
> **Time:** 5–6 hours | **Difficulty:** Intermediate | **Prerequisites:** Lab 03 (`NorthStar_Lab03.pbix`: star schema, marked `dim_date`, hidden keys)
> **Guided lab coverage:** PART 3 Step 3.1 (base measures) and the `Revenue % of Company Total` measure from Step 3.4; PART 6 Optimizations 3–4
> **Produces:** `C:\DataEngineering\sk06\pbix\NorthStar_Lab04.pbix`

## Learning objectives

By the end of this lab you can:

- Choose between a **measure** and a **calculated column**, and explain the cost of each.
- Explain **filter context** (where filters come from and how they reach the fact table) and **row context** (iterators).
- Use `CALCULATE` to change filter context, `REMOVEFILTERS` for "% of total", and `DIVIDE` for safe division.
- Organise measures in a measure table with display folders, formats and descriptions.
- Reconcile measures with an independent SQL query and with a DAX query.

**Key terms**

| Term | Meaning |
|---|---|
| **Measure** | A named DAX formula evaluated at query time in the current filter context; never stored. "If it aggregates, it's a measure." |
| **Calculated column** | A DAX formula evaluated once per row at refresh and stored in the model (costs memory). Prefer Power Query for row-level columns. |
| **Filter context** | The set of filters active for one evaluation: visual rows/columns, slicers, page/report filters, cross-highlighting — propagated along relationships to the facts. |
| **Row context** | "The current row" inside a calculated column or an iterator (`SUMX`, `AVERAGEX`, `FILTER`). Row context does **not** filter the model by itself. |
| **CALCULATE** | Evaluates an expression under a **modified** filter context. The only common way to change context. |
| **REMOVEFILTERS** | A CALCULATE modifier that removes filters from a table or columns (the modern, clearer name for that use of `ALL`). |
| **DIVIDE** | Safe division: returns BLANK (or an alternate) instead of an error when the denominator is 0 or blank. |

### Measures vs calculated columns

| | Calculated column | Measure |
|---|---|---|
| Computed | Once, at refresh — stored per row | At query time — never stored |
| Evaluated in | Row context | Filter context |
| Cost | Model size (RAM) | CPU at query time |
| Use for | Attributes to slice or filter by | Anything you aggregate |

The guided lab's Part 6, Optimization 3, shows the classic mistake: a `margin_pct` *column* stored on 310,024 rows, which is also **wrong** at every total (an average of row percentages is not the total margin %). The measure `Gross Margin %` is smaller *and* correct.

### Filter context in one example

In a matrix with categories on rows and fiscal years on columns, the cell *Grocery × 2026* evaluates `[Net Revenue]` like this:

1. Collect filters: `dim_product[category_name] = "Grocery"`, `dim_date[Fiscal Year] = 2026`.
2. Propagate them along the relationships (dimension → fact) to `fact_sales`.
3. Evaluate `SUM(fact_sales[net_revenue])` over the surviving rows.

The formula never changes — the context does. The grand total cell simply has (almost) no filters.

## Steps

### Step 1 — Open a copy and create the measure table

1. `docker compose start` in `C:\DataEngineering\sk06`.
2. Open `NorthStar_Lab03.pbix` → **Save As** `C:\DataEngineering\sk06\pbix\NorthStar_Lab04.pbix`.
3. **Home → Enter data** → leave the empty column, name the table **`_Measures`** → **Load**.

Measures technically live "in" a table. A dedicated table keeps a 27-measure library findable (the underscore sorts it first). After Step 2 you will hide its dummy `Column1`; the table icon then turns into a calculator.

✅ **Checkpoint:** `_Measures` appears at the top of the Data pane.

### Step 2 — The first measure, done properly

Right-click `_Measures` → **New measure**. The formula bar opens. Replace `Measure =` with:

```dax
Net Revenue =
SUM(fact_sales[net_revenue])
```

Press Enter. Then, with the measure selected:

1. **Measure tools → Format** → Currency, 2 decimals.
2. **Model view** → select the measure → **Properties** pane → **Display folder**: `Sales`; **Description**: `Net revenue = quantity × unit price − discount, per order line (the CFO's definition).`
3. Hide `_Measures[Column1]`.

**Why the name has no brackets or prefix:** the guided lab groups measures with **display folders** (`Sales`, `Time Intelligence`, `KPI`, …). A name like `[Sales] Net Revenue` would contain brackets, which DAX uses to *reference* measures — `[Sales] Net Revenue - [Sales] Total Cost` could not be parsed.

**Verify:** add a **Card** with `Net Revenue`, and a slicer with `dim_date[Fiscal Year]` set to **2026**.

**Expected output:** **$11,862,350.26** — exactly the FY2026 control total from Lab 00 Step 9.

**Common mistake:** *New column* and *New measure* sit next to each other. If your "measure" shows a value on every row in Table view, it is a column — delete it and redo.

✅ **Checkpoint:** the card shows $11,862,350.26 for FY2026.

### Step 3 — The rest of the base measures (guided lab Step 3.1)

Create each as a separate measure in `_Measures`, display folder `Sales`. Formats in the table below.

```dax
Total Cost =
SUM(fact_sales[total_cost])
```

```dax
Gross Margin =
[Net Revenue] - [Total Cost]
```

```dax
// WHY DIVIDE not "/": DIVIDE handles division by zero gracefully.
// BLANK() is the correct alternate result — not 0 (which would
// incorrectly suggest a 0% margin rather than no data).
Gross Margin % =
DIVIDE([Gross Margin], [Net Revenue], BLANK())
```

```dax
Order Count =
DISTINCTCOUNT(fact_sales[order_id])
```

```dax
Average Order Value =
DIVIDE([Net Revenue], [Order Count], BLANK())
```

```dax
Quantity Sold =
SUM(fact_sales[quantity])
```

```dax
Return Rate =
VAR TotalOrders    = DISTINCTCOUNT(fact_sales[order_id])
VAR ReturnedOrders =
    CALCULATE(
        DISTINCTCOUNT(fact_sales[order_id]),
        fact_sales[is_returned] = 1
    )
RETURN DIVIDE(ReturnedOrders, TotalOrders, BLANK())
```

| Measure | Format |
|---|---|
| Total Cost, Gross Margin, Average Order Value | Currency, 2 decimals |
| Gross Margin %, Return Rate | Percentage, 2 decimals |
| Order Count, Quantity Sold | Whole number, thousands separator |

**Reading the formulas:**

- `Gross Margin` and `Average Order Value` **reference other measures** (`[Net Revenue]`). Define once, reuse everywhere — change the definition of revenue and every dependent measure follows.
- `Order Count` uses `DISTINCTCOUNT` because the grain is the order **line**: an order with three lines must count once.
- `Return Rate` uses **variables** (`VAR … RETURN`): each is evaluated once and reused (guided lab Part 6, Optimization 4). Inside `CALCULATE`, `fact_sales[is_returned] = 1` adds a filter to the current context — so "returned orders in FY2026 in the Northeast" works automatically.

**Verify** — a **Table** visual with all eight measures, Fiscal Year slicer = 2026:

| Measure | Expected (FY2026) |
|---|---|
| Net Revenue | $11,862,350.26 |
| Total Cost | $7,655,104.68 |
| Gross Margin | $4,207,245.58 |
| Gross Margin % | 35.47% |
| Order Count | 63,879 |
| Average Order Value | $185.70 |
| Quantity Sold | 179,431 |
| Return Rate | 3.93% |

✅ **Checkpoint:** all eight values match.

### Step 4 — See filter context at work

Build a **Matrix**: rows `dim_product[category_name]`, columns `dim_date[Fiscal Year]`, values `Net Revenue`. Clear the Fiscal Year slicer (or delete it).

**Expected output** (Fiscal Years 2024–2026; empty years do not appear):

| category_name | 2024 | 2025 | 2026 |
|---|---|---|---|
| Apparel | $1,556,310.35 | $1,661,332.87 | $1,799,375.91 |
| Electronics | $2,171,036.33 | $2,314,105.39 | $2,611,036.53 |
| Grocery | $1,081,815.83 | $1,185,238.19 | $1,275,814.16 |
| Home & Garden | $2,443,954.21 | $2,748,447.30 | $2,942,530.95 |
| Sports & Outdoors | $2,705,655.90 | $2,873,909.97 | $3,233,592.71 |
| **Total** | **$9,958,772.62** | **$10,783,033.72** | **$11,862,350.26** |

Say out loud, for three cells:

- *Grocery × 2026:* category = Grocery **and** Fiscal Year = 2026, flowing from `dim_product` and `dim_date` into `fact_sales`.
- *Column total 2026:* only Fiscal Year = 2026.
- *Grand total:* no filters — all three years, $32,604,156.60.

If you can narrate all three, you understand filter context better than many working analysts. Keep this matrix: it is your test harness.

✅ **Checkpoint:** the matrix matches the table above.

### Step 5 — Row context: the same revenue with an iterator

`net_revenue` was computed per line in Lab 02. You must also be able to compute row-level logic in DAX when the source is not yours to change. Create a practice measure (display folder `Practice`):

```dax
Net Revenue (X) =
SUMX(
    fact_sales,
    fact_sales[quantity] * fact_sales[unit_price] - fact_sales[discount_amount]
)
```

`SUMX` first receives `fact_sales` **already reduced by the filter context** of the cell, then walks those rows one by one (**row context**), computes the expression per row, and sums. Filter context first, then row context — every time.

**Verify:** add `Net Revenue (X)` to the matrix — identical to `Net Revenue` in every cell. Then delete the practice measure (the guided lab does not need it; the stored column is cheaper at query time).

✅ **Checkpoint:** the two measures agreed in every cell before you deleted the practice one.

### Step 6 — CALCULATE and "% of company total"

Add this measure from the guided lab's Step 3.4 (display folder `Analysis`, format Percentage, 2 decimals):

```dax
// REMOVEFILTERS(dim_store) removes every store/district/region filter,
// so the denominator is the whole company. Filters on other tables —
// the Fiscal Year slicer on dim_date, a category filter — are PRESERVED.
Revenue % of Company Total =
DIVIDE(
    [Net Revenue],
    CALCULATE(
        [Net Revenue],
        REMOVEFILTERS(dim_store)
    ),
    BLANK()
)
```

In a Northeast cell, the numerator sees Northeast; the denominator's `CALCULATE` removes all `dim_store` filters and sees the whole company. The date filter is untouched, so the share is "of the company, in the selected year".

**Verify:** matrix rows `dim_store[region_name]`, values `Net Revenue` and `Revenue % of Company Total`, Fiscal Year slicer = 2026:

| region_name | Net Revenue | Revenue % of Company Total |
|---|---|---|
| Midwest | $3,053,466.54 | 25.74% |
| Northeast | $3,346,490.01 | 28.21% |
| Southeast | $3,103,316.68 | 26.16% |
| West | $2,359,077.03 | 19.89% |
| **Total** | **$11,862,350.26** | **100.00%** |

**Why the whole table, not just two columns?** The original guided lab removed filters only from `region_name` and `store_name`. Prove the difference: add a slicer on `dim_store[district_name]` and pick **New England**. The correct measure shows **8.27%** (New England's $980,819.76 share of the company). A version that removes only region and store filters would keep the district filter in the denominator and show **100%** — the denominator would be New England itself. Clear the district slicer afterwards.

**Common mistake:** `REMOVEFILTERS()` with no argument removes *every* filter, dates too — the % then answers a different question.

✅ **Checkpoint:** the four regions sum to 100% for FY2026, and New England shows 8.27%.

### Step 7 — Reconcile with SQL and with a DAX query

**SQL (independent of Power BI):**

```powershell
docker exec -it sk06-postgres psql -U postgres -d northstar
```

```sql
SELECT
    ROUND(SUM(ol.quantity * ol.unit_price - ol.discount_amount), 2)  AS net_revenue,
    ROUND(SUM(ol.quantity * p.unit_cost), 2)                          AS total_cost,
    COUNT(DISTINCT o.order_id)                                        AS orders,
    COUNT(DISTINCT o.order_id) FILTER (WHERE o.is_returned = 1)       AS returned_orders,
    SUM(ol.quantity)                                                  AS quantity
FROM sales.orders o
JOIN sales.order_lines ol ON ol.order_id = o.order_id
JOIN products.products p  ON p.product_id = ol.product_id
WHERE o.order_date BETWEEN DATE '2025-07-01' AND DATE '2026-06-30';
```

**Expected output:**

```
 net_revenue | total_cost | orders | returned_orders | quantity
-------------+------------+--------+-----------------+----------
 11862350.26 | 7655104.68 |  63879 |            2509 |   179431
(1 row)
```

2,509 / 63,879 = 3.93% — the `Return Rate`. Quit with `\q`.

**DAX query view** (left edge of Power BI Desktop; in older versions use DAX Studio from daxstudio.org): paste and **Run**:

```dax
EVALUATE
SUMMARIZECOLUMNS(
    dim_date[Fiscal Year],
    "Net Revenue", [Net Revenue],
    "Order Count", [Order Count],
    "Gross Margin %", [Gross Margin %]
)
ORDER BY dim_date[Fiscal Year]
```

**Expected output:** three rows — 2024: 9,958,772.62 / 54,446; 2025: 10,783,033.72 / 58,745; 2026: 11,862,350.26 / 63,879 — matching Lab 00's control totals. (Years with no sales are dropped because every measure is blank.)

✅ **Checkpoint:** SQL, DAX query and visuals agree to the cent.

### Step 8 — Document and save

1. Give every measure a one-sentence **Description** (Model view → Properties). In the Service, descriptions appear as tooltips for report users.
2. Check the display folders: `Sales` (8 measures), `Analysis` (1).
3. **Ctrl+S**.

✅ **Checkpoint:** `_Measures` shows folders `Analysis` and `Sales`; every measure is formatted and described.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| *The syntax for 'Net' is incorrect* | Measure name with brackets, e.g. `[Sales] Net Revenue` | Use plain names and display folders. |
| Measure shows the same value for every region | Region filter cannot reach the fact | Check Model view: `dim_store` → `fact_sales` relationship must be active, single direction, on `store_id`. |
| Net Revenue ≠ SQL control total | Slicer on a different year, or the data was regenerated | Check the slicer; refresh the model; rerun the SQL. |
| Order Count far too high | `COUNT` or `COUNTROWS` used instead of `DISTINCTCOUNT` | Orders have several lines — use `DISTINCTCOUNT(fact_sales[order_id])`. |
| Gross Margin % shows 3547% | Format applied twice or measure multiplied by 100 | Keep the formula a fraction; set the format to Percentage. |
| A "measure" appears as a column in Table view | Created with *New column* | Delete it; right-click `_Measures` → **New measure**. |
| `Revenue % of Company Total` is 100% everywhere | `CALCULATE` missing, or `REMOVEFILTERS` on the wrong table | Compare with the code above. |
| DAX query view not visible | Older Power BI Desktop | Update Power BI Desktop, or use DAX Studio. |

## Key takeaways

- If it aggregates, it is a measure. Build small, named, formatted, documented measures and compose them (`Gross Margin` reuses `Net Revenue` and `Total Cost`).
- Filter context is assembled per visual cell and flows from dimensions to facts; row context exists inside iterators and calculated columns.
- `CALCULATE` changes filter context. `REMOVEFILTERS(dim_store)` gives "% of company" that still respects the date slicer.
- `DIVIDE` with `BLANK()` keeps "no data" honest; variables avoid double evaluation.
- Trust nothing you have not reconciled: FY2026 Net Revenue $11,862,350.26, 63,879 orders, Gross Margin 35.47%, Return Rate 3.93%.

**Interview check:** *What does CALCULATE do?* — It evaluates an expression under a modified filter context: its filter arguments add, replace or remove filters before evaluation; % of total and time intelligence are CALCULATE with specific filters. *SUM vs SUMX?* — SUM aggregates one column; SUMX iterates a table in row context, evaluates an expression per row, then sums.

## Next

[Lab 05 — Advanced DAX & Time Intelligence](Lab_05_Advanced_DAX_and_Time_Intelligence.md): fiscal YTD with `"6/30"`, prior year, YoY, moving averages, targets with `USERELATIONSHIP`, the RAG status, ship-date revenue and the 80/20 rule.
