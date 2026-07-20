# Lab 02 — Joins, Aggregations & Window Functions

> **Scenario:** FreshMart's head of sales wants answers that live in *multiple* tables: revenue by region, top customers per store, month-over-month growth. Time to combine tables and slice data like an analyst.
>
> **Time:** 4–6 hours | **Difficulty:** Beginner → Intermediate | **Prerequisite:** Lab 01 completed (the `freshmart` database with 5 tables loaded)

---

## 1. Environment Setup

Nothing new to install. Verify your Lab 01 environment still works:

```powershell
psql -U postgres -d freshmart -c "SELECT COUNT(*) FROM orders;"
```

**Expected output:** a count around `50000`.

**If it fails:**
- `psql: error: connection ... refused` → the PostgreSQL Windows service is stopped. Open Services (`Win+R` → `services.msc`), find `postgresql-x64-16` (or your version), right-click → Start. Lab 00 covers this in detail.
- `FATAL: database "freshmart" does not exist` → redo Lab 01 Step 1.
- `relation "orders" does not exist` → you're connected to the wrong database, or Lab 01's script wasn't run. Rerun `C:\dataeng\sk02\sql\01_create_tables.sql`.

Create today's script file so every query you write is saved:

```powershell
New-Item -ItemType File -Path C:\dataeng\sk02\sql\02_joins_windows.sql -Force
```

---

## 2. Business Context

So far every question you answered used **one table**. Real business questions almost never do:

- *"Revenue by region"* needs `order_lines` (money) + `orders` (which order) + `stores` (region).
- *"Which Premium customers haven't ordered this month?"* needs `customers` + `orders` — including customers with **no** orders, which a naive combination silently drops.
- *"Month-over-month growth"* needs each month's revenue **and** the previous month's, side by side in one row.

**Who consumes this:** finance (revenue reports), marketing (customer segmentation), operations (store performance). **What happens if it fails:** the classic disaster is a join that *duplicates* rows — revenue reports that show double the real number. Companies have restated earnings over exactly this bug. You will create that bug deliberately in Step 4 so you can recognize it forever.

---

## 3. Concept Explanation

### Joins — combining tables row by row

A **JOIN** matches rows from two tables using a condition (usually foreign key = primary key).

| Join type | Keeps | Typical use |
|---|---|---|
| `INNER JOIN` | Only rows that match on both sides | Orders with their customers |
| `LEFT JOIN` | All left rows; NULLs where right has no match | *All* customers, even those with no orders |
| `RIGHT JOIN` | Mirror of LEFT (rarely used — just swap the tables) | — |
| `FULL OUTER JOIN` | Everything from both sides | Reconciling two systems |
| `CROSS JOIN` | Every combination (no condition) | Generating date × store grids |

**Why joins exist:** normalized OLTP databases split data across tables to avoid duplication (one customer row, many order rows). Joins reassemble it on demand. The alternative — one giant table — wastes space and makes updates error-prone; we'll deliberately build such "denormalized" tables later, but in the *warehouse*, on purpose, in Lab 04.

### Aggregation — collapsing many rows into one

`GROUP BY` buckets rows and applies **aggregate functions** (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) per bucket. `HAVING` filters *buckets* (after grouping); `WHERE` filters *rows* (before grouping). This ordering — FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY — is the execution order you learned in Lab 01.

### CTEs — naming intermediate results

A **CTE** (Common Table Expression, `WITH name AS (...)`) gives a subquery a name so complex logic reads top-to-bottom like a recipe instead of inside-out like nested parentheses. Alternatives: nested subqueries (harder to read), temp tables (persist across statements — useful in ETL, overkill here).

### Window functions — aggregate context without collapsing rows

A **window function** computes something over a group of rows (*the window*) but keeps every row. `SUM(x) OVER (PARTITION BY region ORDER BY month)` gives each row a running regional total *without* a GROUP BY.

- `ROW_NUMBER()` / `RANK()` / `DENSE_RANK()` — rank rows within a partition (top-N per group).
- `LAG()` / `LEAD()` — reach into the previous/next row (month-over-month deltas).
- Frames (`ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`) — moving averages.

**Real-world example:** every "top 10 products per store" dashboard, every retention cohort analysis, every running total — window functions. Before they existed (SQL:2003), these required painful self-joins that were slow and buggy.

---

## 4. Step-by-Step Implementation

Work in pgAdmin's Query Tool or psql, and save every statement to `02_joins_windows.sql`.

### Step 1 — Your first INNER JOIN

**What/why:** attach store info to each order.

```sql
SELECT o.order_id, o.order_ts, s.store_name, s.region
FROM orders o
INNER JOIN stores s ON s.store_id = o.store_id
LIMIT 10;
```

The `o` and `s` are **table aliases** — short names so you can disambiguate columns that exist in both tables.

**Expected output:** 10 rows, each order now showing its store name and region.
**Verify:** `SELECT COUNT(*) FROM orders INNER JOIN stores USING (store_id);` returns exactly the orders count (~50,000) — every order has exactly one store, so the join neither drops nor duplicates rows.
**Common mistake:** forgetting the `ON` clause. PostgreSQL then errors (`syntax error`) — but if you write `CROSS JOIN` by accident (comma-separated tables, no WHERE) you get 50,000 × 8 = 400,000 rows silently. Always sanity-check row counts after a join.
**Troubleshooting:** `column reference "store_id" is ambiguous` → both tables have that column; prefix it: `o.store_id`.

### Step 2 — Revenue by region (join + GROUP BY)

**What/why:** the report finance actually wants. Note we exclude cancelled orders — a business rule you must ask about, never assume.

```sql
SELECT
    s.region,
    COUNT(DISTINCT o.order_id)      AS orders,
    SUM(ol.line_amount)             AS revenue,
    ROUND(AVG(ol.line_amount), 2)   AS avg_line_value
FROM orders o
JOIN order_lines ol ON ol.order_id = o.order_id
JOIN stores s       ON s.store_id  = o.store_id
WHERE o.status = 'completed'
GROUP BY s.region
ORDER BY revenue DESC;
```

**Expected output:** 4 rows (North/South/East/West) with revenue in the low millions each (your random data varies).
**Verify:** the sum of the 4 `revenue` values should equal `SELECT SUM(line_amount) FROM order_lines ol JOIN orders o USING (order_id) WHERE o.status='completed';`
**Common mistake:** `COUNT(o.order_id)` instead of `COUNT(DISTINCT o.order_id)` — because each order has 1–5 lines, the join *multiplies* order rows, and plain COUNT counts lines, not orders.
**Troubleshooting:** `column "s.region" must appear in the GROUP BY clause` → every non-aggregated SELECT column must be in GROUP BY. Add it.

### Step 3 — LEFT JOIN: customers who never ordered

**What/why:** marketing wants to re-engage inactive signups. INNER JOIN would *hide* them — the rows you need are precisely the non-matches.

```sql
SELECT c.customer_id, c.full_name, c.signup_date
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL
ORDER BY c.signup_date;
```

**Expected output:** a small set of customers (random data — possibly a few dozen) with no orders.
**Verify:** `(customers with orders) + (this result's count) = 2000`. Check with `SELECT COUNT(DISTINCT customer_id) FROM orders;`
**Common mistake — the classic LEFT JOIN killer:** putting a right-table filter in WHERE, e.g. `WHERE o.status = 'completed'`. NULL status rows fail that test, so WHERE silently converts your LEFT JOIN into an INNER JOIN. Right-table filters belong **in the ON clause**: `ON o.customer_id = c.customer_id AND o.status = 'completed'`.
**Troubleshooting:** zero rows returned → your random data may have every customer ordering. Verify with the count check above; the logic is still correct.

### Step 4 — Create the fanout bug on purpose

**What/why:** **join fanout** = when the join key is not unique on one side, rows multiply. Feel it once, on purpose.

```sql
-- WRONG on purpose: revenue "by order" joining through order_lines twice
SELECT SUM(ol.line_amount) AS looks_like_revenue
FROM orders o
JOIN order_lines ol  ON ol.order_id = o.order_id
JOIN order_lines ol2 ON ol2.order_id = o.order_id;   -- second join multiplies!
```

**Expected output:** a number several times larger than real revenue. Each order's lines got multiplied by its own line count.
**Verify (the professional habit):** before joining, check key uniqueness: `SELECT order_id, COUNT(*) FROM order_lines GROUP BY order_id HAVING COUNT(*) > 1 LIMIT 5;` If the key repeats, joining on it fans out.
**Common mistake in real life:** joining a fact table to a "dimension" that secretly has duplicate keys (e.g., two rows per product after a bad SCD load — foreshadowing Lab 05).
**Fix pattern:** pre-aggregate in a CTE so the join key becomes unique:

```sql
WITH order_totals AS (
    SELECT order_id, SUM(line_amount) AS order_total
    FROM order_lines
    GROUP BY order_id                 -- now order_id is unique
)
SELECT SUM(ot.order_total) AS real_revenue
FROM orders o
JOIN order_totals ot ON ot.order_id = o.order_id
WHERE o.status = 'completed';
```

### Step 5 — Monthly revenue with a CTE

**What/why:** the building block for the window-function steps. `DATE_TRUNC('month', ts)` rounds a timestamp down to the first of its month.

```sql
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', o.order_ts)::date AS month,
        SUM(ol.line_amount)                   AS revenue
    FROM orders o
    JOIN order_lines ol ON ol.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT * FROM monthly ORDER BY month;
```

**Expected output:** ~18 rows, one per month from 2024-06 onward.
**Verify:** revenues sum to the same total as Step 2's verification query.
**Common mistake:** `GROUP BY 1` means "group by the first SELECT column" — handy, but if you reorder columns later the query silently changes meaning. In production scripts, name the column.

### Step 6 — Month-over-month growth with LAG

**What/why:** the CFO's favorite chart. `LAG(revenue) OVER (ORDER BY month)` fetches the previous row's revenue.

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', o.order_ts)::date AS month,
           SUM(ol.line_amount) AS revenue
    FROM orders o
    JOIN order_lines ol ON ol.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month)                        AS prev_month,
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
          / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 1) AS growth_pct
FROM monthly
ORDER BY month;
```

**Expected output:** first row has NULL `prev_month`/`growth_pct` (nothing before it); later rows show small ± percentages.
**Verify:** pick any row; its `prev_month` must equal the row above's `revenue`.
**Common mistake:** dividing by `LAG(...)` without `NULLIF` — a zero-revenue month crashes with `division by zero`. `NULLIF(x, 0)` returns NULL instead, and NULL propagates safely.
**Troubleshooting:** `window function ... requires an OVER clause` → you wrote `LAG(revenue)` bare; every window function needs `OVER (...)`.

### Step 7 — Top 3 customers per store with ROW_NUMBER

**What/why:** "top N per group" is *the* signature window-function interview question.

```sql
WITH customer_store_rev AS (
    SELECT o.store_id, o.customer_id, SUM(ol.line_amount) AS revenue
    FROM orders o
    JOIN order_lines ol ON ol.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY o.store_id, o.customer_id
),
ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY store_id ORDER BY revenue DESC) AS rn
    FROM customer_store_rev
)
SELECT s.store_name, c.full_name, r.revenue, r.rn
FROM ranked r
JOIN stores s    ON s.store_id = r.store_id
JOIN customers c ON c.customer_id = r.customer_id
WHERE r.rn <= 3
ORDER BY s.store_name, r.rn;
```

**Expected output:** 24 rows (8 stores × 3).
**Verify:** every store appears exactly 3 times: wrap it in `SELECT store_name, COUNT(*) ... GROUP BY store_name`.
**Common mistake:** trying `WHERE ROW_NUMBER() OVER (...) <= 3` directly — illegal, because WHERE runs *before* window functions (execution order again). You must rank in a CTE, then filter in the outer query.
**RANK vs ROW_NUMBER:** ties get the same RANK (then a gap); ROW_NUMBER breaks ties arbitrarily. For "top 3 including ties" use `RANK()`.

### Step 8 — Running total and 7-day moving average

**What/why:** running totals show cumulative progress toward targets; moving averages smooth noise.

```sql
WITH daily AS (
    SELECT o.order_ts::date AS day, SUM(ol.line_amount) AS revenue
    FROM orders o
    JOIN order_lines ol ON ol.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY 1
)
SELECT
    day,
    revenue,
    SUM(revenue) OVER (ORDER BY day)                       AS running_total,
    ROUND(AVG(revenue) OVER (
        ORDER BY day
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2)      AS ma_7d
FROM daily
ORDER BY day;
```

**Expected output:** ~540 rows; `running_total` strictly increases; `ma_7d` is smoother than `revenue`.
**Verify:** last row's `running_total` equals total completed revenue (Step 2 check).
**Common mistake:** omitting the `ROWS BETWEEN` frame for the moving average — the default frame (`RANGE ... CURRENT ROW`) is a running average, not a 7-row window.

### Step 9 — Save and finish

Save `02_joins_windows.sql`. Add a note to `C:\dataeng\sk02\notes.md`: what fanout is, and how you'd detect it.

---

## 5. Production Engineering Practices

1. **Row-count assertions around joins.** *Failure story:* a retailer's nightly report joined sales to a product table where a re-run had inserted duplicate SKUs. Revenue doubled overnight; execs made a pricing decision on it. A one-line check — `HAVING COUNT(*) > 1` on the join key — would have caught it. In Lab 06 you'll automate exactly this as a quality check.
2. **CTEs as documentation.** Name CTEs after business concepts (`monthly_revenue`, `active_customers`), not `t1`, `t2`. The next engineer (usually you, in 6 months) reads names, not logic.
3. **Business rules are config, not folklore.** "Exclude cancelled orders" appeared in our WHERE clause. In production, such rules live in one documented place (a view or dbt model), not copy-pasted into 40 reports that drift apart.
4. **NULL-safe arithmetic everywhere.** `NULLIF` for division, `COALESCE` for display. Every untested division is a 2 a.m. page waiting to happen.
5. **LEFT JOIN filters in ON, not WHERE.** Codify it in code review checklists — it is the single most common silent-wrong-results bug in analytical SQL.

---

## 6. Reflection

**What you learned:** joining tables without dropping or multiplying rows, grouping and filtering groups, structuring logic with CTEs, and the window-function toolkit (ranking, LAG/LEAD, running aggregates).

**Why it matters:** these four skills cover roughly 80% of daily analytical SQL, and 100% of SQL interview screens.

### Interview questions

1. **What's the difference between INNER and LEFT JOIN?** *INNER keeps only matching rows; LEFT keeps all left-side rows with NULLs where the right side has no match. Use LEFT when absence is itself information (customers with no orders).*
2. **What is join fanout and how do you prevent it?** *Row multiplication when the join key isn't unique on one side. Prevent by checking key uniqueness before joining and pre-aggregating to the join grain in a CTE.*
3. **WHERE vs HAVING?** *WHERE filters rows before grouping; HAVING filters groups after aggregation. HAVING can reference aggregates; WHERE cannot.*
4. **How do you get the top 3 per group?** *Rank with ROW_NUMBER() OVER (PARTITION BY group ORDER BY metric DESC) in a CTE, then filter rn <= 3 outside — you can't filter on a window function in the same query level.*
5. **ROW_NUMBER vs RANK vs DENSE_RANK?** *ROW_NUMBER: unique sequence, arbitrary tie-break. RANK: ties share a rank, next rank skips. DENSE_RANK: ties share, no skips.*
6. **How do you compute month-over-month growth in SQL?** *Aggregate to monthly grain, then LAG(revenue) OVER (ORDER BY month), with NULLIF to guard the division.*
7. **Why does filtering a LEFT JOIN's right table in WHERE break it?** *NULLs from non-matching rows fail the WHERE predicate, silently turning it into an INNER JOIN. Put right-table filters in ON.*
8. **What's a CTE and when would you use a temp table instead?** *A named subquery for readability, scoped to one statement. Temp tables persist across statements in a session — better for multi-step ETL or when you want to index intermediate results.*

**Common trap:** interviewers ask you to COUNT orders after joining to order lines. If you say `COUNT(*)` instead of `COUNT(DISTINCT order_id)`, you failed the fanout test.

**Next:** [Lab 03 — Advanced SQL & Query Tuning](Lab_03_Advanced_SQL_Query_Tuning.md), where these queries get *fast*.
