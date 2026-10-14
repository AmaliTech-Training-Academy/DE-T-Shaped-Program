# Lab 07 — Analytical SQL and Window Functions

> **Goal:** Answer FreshMart's business questions on the star schema with CTEs and window functions — running totals, moving averages, rankings, year-over-year, cohorts and Pareto analysis.
> **Time:** 180–210 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 06
> **Guided lab coverage:** PART 5 (Steps 5.1–5.5) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Query a star schema (fact + dimensions, one hop each) and structure logic with **CTEs** (`WITH`).
- Explain how a **window function** (`… OVER (PARTITION BY … ORDER BY … ROWS BETWEEN …)`) differs from `GROUP BY`.
- Use `SUM(...) OVER ()` for grand totals and **`SUM(SUM(...)) OVER (…)`** to window over grouped results.
- Use `LAG` for period-over-period change and a **self-join** for year-over-year comparison.
- Rank with `ROW_NUMBER`, `RANK`, `DENSE_RANK` and filter "top N per group".
- Build running totals that **reset each year**, and 7/30-day moving averages with explicit **frames**.
- Use `FIRST_VALUE` for cohort retention and cumulative sums for Pareto/ABC classification.

**Window functions in one sentence:** an aggregate (`SUM`, `AVG`, `COUNT`, …) or ranking function computed over a *window* of related rows **without collapsing them** — every input row stays in the output and gets an extra column.

| Clause | Meaning |
|---|---|
| `PARTITION BY x` | Restart the calculation for each value of `x` (like GROUP BY, but rows stay). |
| `ORDER BY y` | Order rows inside each partition (needed for running totals, LAG, rankings). |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | The **frame**: which rows around the current one are included. |

Window functions run **after** `WHERE`, `GROUP BY` and `HAVING` — so you can't filter on them in the same query's `WHERE`; wrap them in a CTE first.

## Steps

All queries run on the warehouse (schema `public`) after Lab 06. Use an interactive session; optionally save your queries as `C:\DataEngineering\sk02\sql\10_analytics.sql`.

### Step 1 — A star query, a grand total and SUM(SUM()) OVER ()

```sql
SELECT ds.region_name,
       SUM(fs.net_revenue)                                                   AS revenue,
       ROUND(100.0 * SUM(fs.net_revenue) / SUM(SUM(fs.net_revenue)) OVER (), 1) AS pct_of_total
FROM fact_sales fs
JOIN dim_store ds ON ds.store_key = fs.store_key
JOIN dim_date  dd ON dd.date_key  = fs.date_key
WHERE dd.year_number = 2025
GROUP BY ds.region_name
ORDER BY revenue DESC;
```

**Expected output:**

```
 region_name |  revenue   | pct_of_total
-------------+------------+--------------
 West        | 1914881.08 |         25.2
 Midwest     | 1903697.76 |         25.0
 Southeast   | 1901636.85 |         25.0
 Northeast   | 1879383.57 |         24.7
(4 rows)
```

Two joins replace the six hierarchy joins of Lab 02 — the region is already *in* `dim_store`. `SUM(fs.net_revenue)` is the per-region aggregate; `SUM(…) OVER ()` then sums those four results (an empty `OVER ()` = the whole result set), giving the grand total on every row.

✅ **Checkpoint:** the percentages add up to ~100.

### Step 2 — CTEs and LAG: month-over-month growth

```sql
WITH monthly AS (
    SELECT dd.year_month, SUM(fs.net_revenue) AS revenue
    FROM fact_sales fs
    JOIN dim_date dd ON dd.date_key = fs.date_key
    WHERE dd.year_number = 2025
    GROUP BY dd.year_month
)
SELECT year_month,
       revenue,
       LAG(revenue) OVER (ORDER BY year_month)                       AS prev_month,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY year_month))
             / NULLIF(LAG(revenue) OVER (ORDER BY year_month), 0), 1) AS growth_pct
FROM monthly
ORDER BY year_month
LIMIT 3;
```

**Expected output:**

```
 year_month |  revenue  | prev_month | growth_pct
------------+-----------+------------+------------
 2025-01    | 647377.72 |            |
 2025-02    | 587135.26 |  647377.72 |       -9.3
 2025-03    | 656157.76 |  587135.26 |       11.8
(3 rows)
```

A **CTE** (`WITH name AS (…)`) names an intermediate result so the logic reads top to bottom. `LAG(x)` fetches `x` from the previous row in the window order (`LEAD` fetches the next). The first row has no predecessor → NULL. `NULLIF(…, 0)` prevents a division-by-zero error.

✅ **Checkpoint:** February shows −9.3 %.

### Step 3 — Year-over-year with a self-join (guided Step 5.2)

`LAG` compares with the *previous row*; for "same month last year" it is clearer to join a CTE to itself:

```sql
WITH monthly_revenue AS (
    SELECT
        ds.store_name,
        ds.region_name,
        dd.year_number,
        dd.month_number,
        SUM(fs.net_revenue) AS revenue
    FROM      fact_sales fs
    JOIN dim_date  dd ON fs.date_key  = dd.date_key
    JOIN dim_store ds ON fs.store_key = ds.store_key
    WHERE dd.year_number IN (2024, 2025)
    GROUP BY ds.store_name, ds.region_name, dd.year_number, dd.month_number
)
SELECT
    cy.store_name,
    cy.region_name,
    cy.month_number,
    cy.revenue                                           AS cy_revenue,
    py.revenue                                           AS py_revenue,
    ROUND(
        (cy.revenue - COALESCE(py.revenue, 0))
        / NULLIF(py.revenue, 0) * 100,
        2
    )                                                    AS yoy_growth_pct
FROM monthly_revenue cy
LEFT JOIN monthly_revenue py
    ON  cy.store_name   = py.store_name
    AND cy.month_number = py.month_number
    AND py.year_number  = cy.year_number - 1
WHERE cy.year_number = 2025
ORDER BY cy.region_name, cy.store_name, cy.month_number
LIMIT 3;
```

**Expected output:**

```
 store_name  | region_name | month_number | cy_revenue | py_revenue | yoy_growth_pct
-------------+-------------+--------------+------------+------------+----------------
 Chicago #19 | Midwest     |            1 |   10981.36 |    5675.91 |          93.47
 Chicago #19 | Midwest     |            2 |    7839.39 |    3646.41 |         114.99
 Chicago #19 | Midwest     |            3 |   12648.78 |    5586.84 |         126.40
(3 rows)
```

`cy` (current year) and `py` (prior year) are two aliases of the same CTE. The `LEFT JOIN` keeps a 2025 month even if the store had no sales in that month of 2024 (then `py_revenue` is NULL). Without `LIMIT` the query returns 720 rows (60 stores × 12 months).

✅ **Checkpoint:** 93.47 % growth for Chicago #19 in January.

### Step 4 — Rankings: top 3 stores per region

```sql
WITH store_revenue AS (
    SELECT ds.region_name, ds.store_name, SUM(fs.net_revenue) AS revenue
    FROM fact_sales fs
    JOIN dim_store ds ON ds.store_key = fs.store_key
    JOIN dim_date  dd ON dd.date_key  = fs.date_key
    WHERE dd.year_number = 2025
    GROUP BY ds.region_name, ds.store_name
),
ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY region_name ORDER BY revenue DESC) AS rn
    FROM store_revenue
)
SELECT region_name, store_name, revenue, rn
FROM ranked
WHERE rn <= 3
ORDER BY region_name, rn;
```

**Expected output:**

```
 region_name |    store_name    |  revenue  | rn
-------------+------------------+-----------+----
 Midwest     | Chicago #55      | 134011.79 |  1
 Midwest     | Detroit #8       | 133928.07 |  2
 Midwest     | Detroit #56      | 132062.39 |  3
 Northeast   | Boston #50       | 140159.91 |  1
 Northeast   | Boston #14       | 132080.26 |  2
 Northeast   | Philadelphia #15 | 130010.82 |  3
 Southeast   | Charlotte #18    | 135104.93 |  1
 Southeast   | Charlotte #6     | 132611.65 |  2
 Southeast   | Charlotte #42    | 131933.75 |  3
 West        | Seattle #59      | 135870.08 |  1
 West        | Seattle #47      | 135367.91 |  2
 West        | Los Angeles #46  | 132583.53 |  3
(12 rows)
```

"Top N per group" = rank in a CTE, filter outside (you can't write `WHERE ROW_NUMBER() … <= 3`). On ties: `ROW_NUMBER` gives unique numbers (arbitrary order among ties), `RANK` gives ties the same number and then skips (1, 1, 3), `DENSE_RANK` doesn't skip (1, 1, 2).

✅ **Checkpoint:** 12 rows, 3 per region.

### Step 5 — Running totals that reset each year, and moving averages (guided Step 5.4)

```sql
SELECT
    dd.full_date,
    dd.day_name,
    SUM(fs.net_revenue)                                   AS daily_revenue,
    -- YTD cumulative total (resets each year via PARTITION BY)
    SUM(SUM(fs.net_revenue)) OVER (
        PARTITION BY dd.year_number
        ORDER BY dd.full_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                                     AS cumulative_ytd,
    -- 7-day moving average (smooths weekly seasonality)
    ROUND(AVG(SUM(fs.net_revenue)) OVER (
        ORDER BY dd.full_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2)                                                 AS rolling_7day_avg,
    -- 30-day moving average (smooths monthly patterns)
    ROUND(AVG(SUM(fs.net_revenue)) OVER (
        ORDER BY dd.full_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 2)                                                 AS rolling_30day_avg
FROM      fact_sales fs
JOIN dim_date dd ON fs.date_key = dd.date_key
WHERE dd.year_number = 2025
GROUP BY dd.full_date, dd.day_name, dd.year_number
ORDER BY dd.full_date
LIMIT 3;
```

**Expected output:**

```
 full_date  | day_name  | daily_revenue | cumulative_ytd | rolling_7day_avg | rolling_30day_avg
------------+-----------+---------------+----------------+------------------+-------------------
 2025-01-01 | Wednesday |      14973.41 |       14973.41 |         14973.41 |          14973.41
 2025-01-02 | Thursday  |      22447.71 |       37421.12 |         18710.56 |          18710.56
 2025-01-03 | Friday    |      17372.47 |       54793.59 |         18264.53 |          18264.53
(3 rows)
```

How it works: `GROUP BY` first produces one row per day with `SUM(fs.net_revenue)`; the window functions then run over those daily rows — hence `SUM(SUM(...))` and `AVG(SUM(...))`. `PARTITION BY dd.year_number` restarts the running total every 1 January. `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` = the current day plus up to 6 before it (on day 3 only 3 days exist, so both averages are the same). Without an explicit frame, `ORDER BY` defaults to "everything up to the current row" — a running average, not a 7-day one. Without `LIMIT` you get 365 rows.

✅ **Checkpoint:** `cumulative_ytd` on 3 January = 14973.41 + 22447.71 + 17372.47.

### Step 6 — FIRST_VALUE and cohort retention (guided Step 5.3)

A **cohort** = customers whose first purchase fell in the same month. "Period n" = n months after that. Retention % = active customers in period n ÷ active customers in period 0. `FIRST_VALUE(x) OVER (PARTITION BY cohort ORDER BY period …)` fetches the period-0 count onto every row of the cohort.

```sql
-- 5.3 Cohort retention — cohort = month of a customer's FIRST purchase.
--     Grouping by customer_id (natural key) counts purchases recorded
--     against ANY SCD2 version of the customer.
WITH customer_cohorts AS (
    SELECT
        dc.customer_id,
        MIN(DATE_TRUNC('month', dd.full_date))::DATE AS cohort_month
    FROM      fact_sales   fs
    JOIN dim_date     dd ON fs.date_key     = dd.date_key
    JOIN dim_customer dc ON fs.customer_key = dc.customer_key
    GROUP BY dc.customer_id
),
active_by_month AS (
    SELECT DISTINCT
        dc.customer_id,
        DATE_TRUNC('month', dd.full_date)::DATE AS activity_month
    FROM      fact_sales   fs
    JOIN dim_date     dd ON fs.date_key     = dd.date_key
    JOIN dim_customer dc ON fs.customer_key = dc.customer_key
),
cohort_data AS (
    SELECT
        cc.cohort_month,
        (
            (EXTRACT(YEAR  FROM ab.activity_month) * 12
            + EXTRACT(MONTH FROM ab.activity_month))
            -
            (EXTRACT(YEAR  FROM cc.cohort_month) * 12
            + EXTRACT(MONTH FROM cc.cohort_month))
        )::INT                              AS period_number,
        COUNT(DISTINCT cc.customer_id)      AS active_customers
    FROM customer_cohorts cc
    JOIN active_by_month  ab ON cc.customer_id = ab.customer_id
    GROUP BY cc.cohort_month, period_number
)
SELECT
    cohort_month,
    period_number,
    active_customers,
    ROUND(
        active_customers * 100.0
        / FIRST_VALUE(active_customers) OVER (
            PARTITION BY cohort_month
            ORDER BY period_number
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ),
        1
    ) AS retention_pct
FROM cohort_data
WHERE period_number BETWEEN 0 AND 12
ORDER BY cohort_month, period_number;
```

**Expected output** (one row per cohort month and period 0–12; the first four):

```
 cohort_month | period_number | active_customers | retention_pct
--------------+---------------+------------------+---------------
 2023-01-01   |             0 |             1275 |         100.0
 2023-01-01   |             1 |              120 |           9.4
 2023-01-01   |             2 |              164 |          12.9
 2023-01-01   |             3 |              153 |          12.0
```

Notice the joins go through `dim_customer` to `customer_id`: a customer with two SCD2 versions (like customer 500) is still **one** customer here.

✅ **Checkpoint:** period 0 is 100.0 for every cohort.

### Step 7 — Pareto / ABC classification (guided Step 5.5)

Which products make 80% of revenue? Rank products by revenue, compute a **cumulative** share with `SUM(...) OVER (ORDER BY revenue DESC)`, then classify with `CASE`:

```sql
WITH product_revenue AS (
    SELECT
        dp.product_id,
        SUM(fs.net_revenue) AS total_revenue
    FROM      fact_sales   fs
    JOIN dim_product dp ON fs.product_key = dp.product_key
    JOIN dim_date    dd ON fs.date_key    = dd.date_key
    WHERE dd.year_number = 2025
    GROUP BY dp.product_id
),
ranked AS (
    SELECT
        cur.product_name,
        cur.category_name,
        pr.total_revenue,
        SUM(pr.total_revenue) OVER (ORDER BY pr.total_revenue DESC, pr.product_id) AS cumulative_revenue,
        SUM(pr.total_revenue) OVER ()                                            AS grand_total,
        ROW_NUMBER()          OVER (ORDER BY pr.total_revenue DESC, pr.product_id) AS rank,
        COUNT(*)              OVER ()                                            AS total_products
    FROM product_revenue pr
    JOIN dim_product cur ON cur.product_id = pr.product_id AND cur.is_current = TRUE
)
SELECT
    rank,
    product_name,
    category_name,
    ROUND(total_revenue, 2)                                     AS revenue,
    ROUND(cumulative_revenue / grand_total * 100, 1)            AS cumulative_pct,
    CASE
        WHEN cumulative_revenue / grand_total <= 0.80 THEN 'A — Top 80%'
        WHEN cumulative_revenue / grand_total <= 0.95 THEN 'B — Next 15%'
        ELSE                                                'C — Bottom 5%'
    END                                                         AS abc_class,
    ROUND(rank * 100.0 / total_products, 1)                    AS product_percentile
FROM ranked
ORDER BY rank
LIMIT 3;
```

**Expected output:**

```
 rank |     product_name      | category_name | revenue  | cumulative_pct |  abc_class  | product_percentile
------+-----------------------+---------------+----------+----------------+-------------+--------------------
    1 | Summit Nuts 907       | Snacks        | 14619.98 |            0.2 | A — Top 80% |                0.1
    2 | Sunny Kitchenware 912 | Household     | 14525.66 |            0.4 | A — Top 80% |                0.2
    3 | Summit Water 1117     | Beverages     | 14412.67 |            0.6 | A — Top 80% |                0.3
(3 rows)
```

Two design points: revenue is summed per **`product_id`** (all SCD2 versions together) and then labelled with the **current** version's name and category — otherwise product 7 would appear twice. The tie-breaker `pr.product_id` in the window `ORDER BY` makes ranks deterministic.

Summarise the classes: replace the final `SELECT … LIMIT 3` with

```sql
SELECT CASE WHEN cumulative_revenue / grand_total <= 0.80 THEN 'A'
            WHEN cumulative_revenue / grand_total <= 0.95 THEN 'B'
            ELSE 'C' END AS abc_class,
       COUNT(*)          AS products
FROM ranked
GROUP BY 1
ORDER BY 1;
```

**Expected output:**

```
 abc_class | products
-----------+----------
 A         |      684
 B         |      277
 C         |      239
(3 rows)
```

(Sample data is much flatter than real retail, where roughly 20% of products make 80% of revenue.)

✅ **Checkpoint:** 684 + 277 + 239 = 1200 products.

### Step 8 — The 45-minute report on the star (guided Step 5.1)

Run the guided lab's Step 5.1 query (the star-schema version of Lab 02's Step 10). Count its rows the same way:

```sql
SELECT COUNT(*) AS report_rows FROM (
    SELECT ds.region_name, ds.store_name, dp.category_name, dp.product_name, dd.year_month,
           COUNT(DISTINCT fs.order_id), SUM(fs.quantity), SUM(fs.gross_revenue),
           SUM(fs.discount_amount), SUM(fs.net_revenue)
    FROM      fact_sales   fs
    JOIN dim_date     dd ON fs.date_key    = dd.date_key
    JOIN dim_store    ds ON fs.store_key   = ds.store_key
    JOIN dim_product  dp ON fs.product_key = dp.product_key
    WHERE dd.year_number = 2025
    GROUP BY ds.region_name, ds.store_name, dp.category_name, dp.product_name, dd.year_month
) AS t;
```

**Expected output:**

```
 report_rows
-------------
      115705
(1 row)
```

115,705 instead of Lab 02's 122,481: the warehouse applies the business rule "only completed orders are revenue" once, at load time — every report inherits it.

✅ **Checkpoint:** 115705 rows from 3 joins instead of 7.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `window functions are not allowed in WHERE` | Filtering on a window result in the same query | Compute it in a CTE/subquery, filter outside. |
| `aggregate function calls cannot contain window function calls` | Wrote `SUM(SUM(x) OVER …)` (wrong nesting) | The window goes outside: `SUM(SUM(x)) OVER (…)`. |
| `column "x" must appear in the GROUP BY clause` inside a window query | A window's `PARTITION BY`/`ORDER BY` uses a column not in `GROUP BY` | Add it to `GROUP BY` (e.g. `dd.year_number` in Step 5). |
| Moving average equals the running average | Frame missing | Add `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`. |
| A product appears twice in Pareto | Grouped by `product_key` / name instead of `product_id` | Group by `dp.product_id`, label with the current version. |
| Numbers differ from the expected output | Day-2 script not run, or warehouse rebuilt | Lab 06 Steps 6–7 must be complete (2025 results don't change, but cohort/Pareto labels do). |

## Key takeaways

- Star queries: one join per dimension, business rules already applied.
- Window = aggregate without collapsing rows; `PARTITION BY` restarts, `ORDER BY` orders, `ROWS BETWEEN` frames.
- `SUM(SUM(x)) OVER (…)` windows over grouped results; `SUM(x) OVER ()` is the grand total.
- `LAG` for previous period, self-join for same-period-last-year, `ROW_NUMBER` + CTE for top-N, `FIRST_VALUE` for cohort baselines, cumulative `SUM … OVER (ORDER BY …)` for Pareto.

## Next

[Lab 08 — Performance: Indexes, Materialized Views and Partitioning](Lab_08_Performance_Indexes_MVs_Partitioning.md): make these queries fast for dashboards.
