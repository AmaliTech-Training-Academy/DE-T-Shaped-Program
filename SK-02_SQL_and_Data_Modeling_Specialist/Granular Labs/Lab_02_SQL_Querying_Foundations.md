# Lab 02 — SQL Querying Foundations

> **Goal:** Query FreshMart's source system confidently: filter rows, handle dates and NULLs, join across the normalized hierarchies, aggregate, and spot join fan-out.
> **Time:** 150–180 min | **Difficulty:** Beginner | **Prerequisites:** Lab 01
> **Guided lab coverage:** PART 1 Step 1.2 (reading the 8-join sales report) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Write `SELECT … FROM … WHERE … ORDER BY … LIMIT` and explain SQL's **logical execution order**.
- Filter dates safely with half-open ranges, and use `EXTRACT`, `DATE_TRUNC` and `::` casts.
- Handle `NULL` correctly (`IS NULL`, `COALESCE`, `COUNT(col)` vs `COUNT(*)`).
- Use `CASE` and `FILTER (WHERE …)` for conditional logic and conditional counts.
- Join tables with `INNER JOIN` and `LEFT JOIN`, including multi-hop joins through a snowflaked hierarchy.
- Aggregate with `GROUP BY` / `HAVING`, and recognise **fan-out** (why `COUNT(DISTINCT …)` exists).
- Read and run the guided lab's Step 1.2 report query.

**The one mental model to memorise** — SQL is *written* in one order but *evaluated* in another:

```
Written:   SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
Evaluated: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

That is why a column alias made in `SELECT` can be used in `ORDER BY` but not in `WHERE`.

## Steps

Work in an interactive session (`docker exec -it sk02-postgres psql -U postgres -d freshmart`). Optionally collect your queries in `C:\DataEngineering\sk02\sql\lab02_queries.sql`.

### Step 1 — SELECT, column aliases, ORDER BY, LIMIT

```sql
SELECT store_id, store_name, store_type, city, state
FROM source_oltp.stores
ORDER BY store_id
LIMIT 5;
```

**Expected output:**

```
 store_id |   store_name    | store_type  |     city     | state
----------+-----------------+-------------+--------------+-------
        1 | New York #1     | Supermarket | New York     | NY
        2 | Boston #2       | Supermarket | Boston       | MA
        3 | Philadelphia #3 | Express     | Philadelphia | PA
        4 | Atlanta #4      | Supermarket | Atlanta      | GA
        5 | Miami #5        | Supermarket | Miami        | FL
(5 rows)
```

`SELECT *` is fine for a quick peek at a small table, but always name columns in saved scripts, and always `LIMIT` when exploring a big table (`order_items` has 300,235 rows).

✅ **Checkpoint:** 5 rows, in `store_id` order.

### Step 2 — Filtering with WHERE, AND/OR, IN, BETWEEN, LIKE

```sql
-- All orders of customer 42, oldest first
SELECT order_id, order_date, status
FROM source_oltp.orders
WHERE customer_id = 42
ORDER BY order_date, order_id;

-- Set membership + numeric range: Dairy/Bakery products priced 10–20
SELECT COUNT(*) AS products
FROM source_oltp.products p
JOIN source_oltp.product_subcats ps ON ps.subcategory_id = p.subcategory_id
JOIN source_oltp.product_categories pc ON pc.category_id = ps.category_id
WHERE pc.category_name IN ('Dairy', 'Bakery')
  AND p.list_price BETWEEN 10 AND 20;

-- Pattern matching: % = any characters
SELECT COUNT(*) AS emmas_j
FROM source_oltp.customers
WHERE first_name = 'Emma' AND last_name LIKE 'J%';
```

**Expected output:**

```
 order_id | order_date |  status
----------+------------+-----------
    61340 | 2024-06-17 | completed
    58066 | 2024-11-23 | completed
    76339 | 2025-01-18 | cancelled
    39947 | 2025-03-31 | completed
    66885 | 2025-05-21 | completed
    13616 | 2025-12-31 | completed
(6 rows)

 products
----------
       87
(1 row)

 emmas_j
---------
     131
(1 row)
```

(The middle query already uses joins — Step 6 explains them.) **Strings use single quotes** (`'Dairy'`); double quotes are for identifiers — `WHERE first_name = "Emma"` fails with `column "Emma" does not exist`.

✅ **Checkpoint:** customer 42 has 6 orders, one cancelled.

### Step 3 — Dates: half-open ranges, EXTRACT, DATE_TRUNC, casts

```sql
-- Orders placed in January 2025 (order_date is a DATE)
SELECT COUNT(*) AS jan_orders
FROM source_oltp.orders
WHERE order_date >= DATE '2025-01-01' AND order_date < DATE '2025-02-01';

-- The classic trap: BETWEEN on a TIMESTAMP column
SELECT
  COUNT(*) FILTER (WHERE updated_at BETWEEN '2025-01-01' AND '2025-01-31')         AS with_between,
  COUNT(*) FILTER (WHERE updated_at >= '2025-01-01' AND updated_at < '2025-02-01') AS half_open
FROM source_oltp.orders;

-- Orders per year
SELECT EXTRACT(YEAR FROM order_date)::int AS order_year,
       COUNT(*)                            AS orders
FROM source_oltp.orders
GROUP BY order_year
ORDER BY order_year;

-- DATE_TRUNC rounds down to the start of the month (the report's sales_month)
SELECT DATE_TRUNC('month', DATE '2025-03-31')::date AS month_start;
```

**Expected output:**

```
 jan_orders
------------
       3756
(1 row)

 with_between | half_open
--------------+-----------
         3635 |      3756
(1 row)

 order_year | orders
------------+--------
       2023 |  21219
       2024 |  34775
       2025 |  44006
(3 rows)

 month_start
-------------
 2025-03-01
(1 row)
```

`'2025-01-31'` as a timestamp means *midnight at the start of* Jan 31, so `BETWEEN` silently drops almost the whole last day (121 orders here). Always use `>= start AND < next_start` for dates and timestamps. `::int` / `::date` are PostgreSQL casts (standard SQL: `CAST(x AS int)`). `GROUP BY order_year` works because PostgreSQL lets `GROUP BY` use a SELECT alias.

✅ **Checkpoint:** `with_between` < `half_open`, and the three yearly counts add up to 100,000.

### Step 4 — NULL: unknown, not zero

```sql
SELECT
  COUNT(*) FILTER (WHERE email IS NULL) AS no_email_correct,
  COUNT(*) FILTER (WHERE email = NULL)  AS no_email_wrong,
  COUNT(email)                          AS count_email_column,
  COUNT(*)                              AS all_customers
FROM source_oltp.customers;

SELECT customer_id, first_name, last_name, COALESCE(email, '(no email on file)') AS contact
FROM source_oltp.customers
WHERE email IS NULL
ORDER BY customer_id
LIMIT 3;

SELECT NULL = NULL AS equals, NULL IS NOT DISTINCT FROM NULL AS null_safe_equals;
```

**Expected output:**

```
 no_email_correct | no_email_wrong | count_email_column | all_customers
------------------+----------------+--------------------+---------------
             2957 |              0 |              17043 |         20000
(1 row)

 customer_id | first_name | last_name |      contact
-------------+------------+-----------+--------------------
           8 | Isabella   | Hernandez | (no email on file)
          10 | Mia        | Johnson   | (no email on file)
          11 | Ethan      | Rodriguez | (no email on file)
(3 rows)

 equals | null_safe_equals
--------+------------------
        | t
(1 row)
```

Any comparison with NULL is *unknown*, and `WHERE` keeps only *true* — so `= NULL` matches nothing. `COUNT(column)` skips NULLs; `COUNT(*)` counts rows. `IS DISTINCT FROM` / `IS NOT DISTINCT FROM` are the NULL-safe comparisons — the SCD Type 2 ETL in Lab 05 depends on them.

✅ **Checkpoint:** 2957 + 17043 = 20000.

### Step 5 — CASE and conditional aggregation

```sql
SELECT
    EXTRACT(YEAR FROM order_date)::int                         AS order_year,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END)      AS completed_case,
    COUNT(*) FILTER (WHERE status = 'completed')               AS completed_filter,
    COUNT(*) FILTER (WHERE status IN ('cancelled','returned')) AS not_revenue
FROM source_oltp.orders
GROUP BY order_year
ORDER BY order_year;
```

**Expected output:**

```
 order_year | completed_case | completed_filter | not_revenue
------------+----------------+------------------+-------------
       2023 |          19879 |            19879 |        1340
       2024 |          32743 |            32743 |        2032
       2025 |          41364 |            41364 |        2642
(3 rows)
```

`CASE` is SQL's if/else; `FILTER (WHERE …)` is PostgreSQL's shorter way to count or sum only some rows. Business rule used from here on: **only `completed` orders are revenue.**

✅ **Checkpoint:** both "completed" columns are identical.

### Step 6 — INNER JOIN through the normalized hierarchies

In 3NF, a product's category is two hops away: `products → product_subcats → product_categories`. A store's region is also two hops away.

```sql
SELECT p.product_id, p.product_name, ps.subcategory_name, pc.category_name, s.supplier_name
FROM source_oltp.products p
JOIN source_oltp.product_subcats    ps ON ps.subcategory_id = p.subcategory_id
JOIN source_oltp.product_categories pc ON pc.category_id    = ps.category_id
JOIN source_oltp.suppliers          s  ON s.supplier_id     = p.supplier_id
WHERE p.product_id = 7;

SELECT sr.region_name, COUNT(*) AS stores
FROM source_oltp.stores s
JOIN source_oltp.store_districts sd ON sd.district_id = s.district_id
JOIN source_oltp.store_regions   sr ON sr.region_id   = sd.region_id
GROUP BY sr.region_name
ORDER BY sr.region_name;
```

**Expected output:**

```
 product_id |  product_name   | subcategory_name | category_name | supplier_name
------------+-----------------+------------------+---------------+---------------
          7 | Summit Yogurt 7 | Yogurt           | Dairy         | Supplier 10
(1 row)

 region_name | stores
-------------+--------
 Midwest     |     15
 Northeast   |     15
 Southeast   |     15
 West        |     15
(4 rows)
```

`p`, `ps`, `pc` are **table aliases**. Every report that needs "category" or "region" must repeat these hops — this is exactly the pain the warehouse removes by **denormalizing** them into one wide `dim_product` / `dim_store` row (Lab 05).

✅ **Checkpoint:** product 7 is a Dairy product.

### Step 7 — LEFT JOIN: keep rows without a match

```sql
SELECT COUNT(*) AS customers_without_orders
FROM source_oltp.customers c
LEFT JOIN source_oltp.orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;

SELECT COUNT(DISTINCT customer_id) AS customers_with_orders
FROM source_oltp.orders;
```

**Expected output:**

```
 customers_without_orders
--------------------------
                      147
(1 row)

 customers_with_orders
-----------------------
                 19853
(1 row)
```

A `LEFT JOIN` keeps every left-side row and fills the right side with NULLs when there is no match; `WHERE o.order_id IS NULL` then keeps exactly the non-matches. **Trap:** a filter on the right table (e.g. `o.status = 'completed'`) placed in `WHERE` turns the LEFT JOIN back into an INNER JOIN — put such filters in the `ON` clause.

✅ **Checkpoint:** 147 + 19853 = 20000.

### Step 8 — GROUP BY, HAVING and revenue

Net revenue of a line = `quantity × unit_price − discount_amount`.

```sql
SELECT pc.category_name,
       SUM(oi.quantity * oi.unit_price - oi.discount_amount) AS net_revenue
FROM source_oltp.orders o
JOIN source_oltp.order_items        oi ON oi.order_id       = o.order_id
JOIN source_oltp.products           p  ON p.product_id      = oi.product_id
JOIN source_oltp.product_subcats    ps ON ps.subcategory_id = p.subcategory_id
JOIN source_oltp.product_categories pc ON pc.category_id    = ps.category_id
WHERE o.status = 'completed'
  AND o.order_date >= DATE '2025-01-01' AND o.order_date < DATE '2026-01-01'
GROUP BY pc.category_name
HAVING SUM(oi.quantity * oi.unit_price - oi.discount_amount) > 1260000
ORDER BY net_revenue DESC;
```

**Expected output:**

```
 category_name | net_revenue
---------------+-------------
 Dairy         |  1346315.82
 Bakery        |  1291100.23
 Produce       |  1268892.12
(3 rows)
```

Remove the `HAVING` line and you get all six categories (Snacks 1257486.98, Household 1252681.23, Beverages 1183122.88). `WHERE` filters **rows before** grouping; `HAVING` filters **groups after** aggregation (it can use `SUM`, `WHERE` cannot). Every non-aggregated column in `SELECT` must appear in `GROUP BY`.

✅ **Checkpoint:** exactly 3 categories pass the `HAVING`.

### Step 9 — Fan-out: why COUNT(DISTINCT) exists

Joining a parent (orders) to its children (order_items) repeats the parent once per child.

```sql
SELECT COUNT(*)                   AS joined_rows,
       COUNT(DISTINCT o.order_id) AS real_orders
FROM source_oltp.orders o
JOIN source_oltp.order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
  AND o.order_date >= DATE '2025-01-01' AND o.order_date < DATE '2026-01-01';
```

**Expected output:**

```
 joined_rows | real_orders
-------------+-------------
      124180 |       41364
(1 row)
```

Counting orders with `COUNT(*)` after this join would triple-count them. That's why the guided report uses `COUNT(DISTINCT o.order_id)`. Before joining, always ask "is the join key unique on the other side?" — if not, rows multiply.

✅ **Checkpoint:** you can explain why the two numbers differ.

### Step 10 — Run the guided lab's Step 1.2 report

You now know every piece of the "45-minute" report. Count how many rows it produces (wrapping a query in `SELECT COUNT(*) FROM (…) AS t` is a handy trick):

```sql
SELECT COUNT(*) AS report_rows FROM (
    SELECT
        s.store_name,
        sr.region_name,
        pc.category_name,
        p.product_name,
        DATE_TRUNC('month', o.order_date) AS sales_month,
        COUNT(DISTINCT o.order_id)                            AS order_count,
        SUM(oi.quantity)                                      AS units_sold,
        SUM(oi.quantity * oi.unit_price)                      AS gross_revenue,
        SUM(oi.discount_amount)                               AS total_discounts,
        SUM(oi.quantity * oi.unit_price - oi.discount_amount) AS net_revenue
    FROM source_oltp.orders             o
    JOIN source_oltp.order_items        oi ON o.order_id       = oi.order_id
    JOIN source_oltp.products           p  ON oi.product_id    = p.product_id
    JOIN source_oltp.product_subcats    ps ON p.subcategory_id = ps.subcategory_id
    JOIN source_oltp.product_categories pc ON ps.category_id   = pc.category_id
    JOIN source_oltp.stores             s  ON o.store_id       = s.store_id
    JOIN source_oltp.store_districts    sd ON s.district_id    = sd.district_id
    JOIN source_oltp.store_regions      sr ON sd.region_id     = sr.region_id
    WHERE o.order_date >= DATE '2025-01-01'
      AND o.order_date <  DATE '2026-01-01'
    GROUP BY s.store_name, sr.region_name, pc.category_name,
             p.product_name, DATE_TRUNC('month', o.order_date)
) AS t;
```

**Expected output:**

```
 report_rows
-------------
      122481
(1 row)
```

Read the inner query aloud: *8 tables, 7 joins, 5 text grouping columns, a year of orders*. Note it does **not** filter `status`, so cancelled and returned orders are counted as revenue — a business-rule bug the warehouse fixes. Lab 03 looks at *how* PostgreSQL executes it.

✅ **Checkpoint:** 122481 rows.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `relation "orders" does not exist` | Schema name missing | Write `source_oltp.orders` (or run `SET search_path = source_oltp, public;` for the session). |
| `column "order_year" does not exist` in WHERE | Alias used before SELECT is evaluated | Repeat the expression in `WHERE`, or use a subquery/CTE. |
| `column "x" must appear in the GROUP BY clause or be used in an aggregate function` | Non-aggregated column missing from GROUP BY | Add it to `GROUP BY` (or aggregate it). |
| `column reference "order_id" is ambiguous` | Column exists in two joined tables | Prefix it with the table alias: `o.order_id`. |
| Counts much bigger than expected after a join | Join fan-out | Check key uniqueness; use `COUNT(DISTINCT …)` or aggregate before joining. |
| psql prompt shows `freshmart-#` | Missing `;` | Type `;` and press Enter. |

## Key takeaways

- Evaluation order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT) explains most "weird" SQL errors.
- Half-open ranges for dates; `IS NULL` / `IS DISTINCT FROM` for NULLs; `FILTER` / `CASE` for conditional aggregates.
- Normalized sources need multi-hop joins for every report — the motivation for a star schema.
- Joins to child tables fan out; count parents with `COUNT(DISTINCT …)`.

## Next

[Lab 03 — Query Plans and Indexes](Lab_03_Query_Plans_and_Indexes.md): use `EXPLAIN ANALYZE` to see *why* the report is slow.
