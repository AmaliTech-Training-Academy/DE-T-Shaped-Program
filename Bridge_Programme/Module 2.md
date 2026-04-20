# MODULE 2: SQL Beyond the Basics

## Why This Module Exists

You know SELECT, WHERE, and maybe basic JOINs. Data engineering SQL is different. Data engineers write queries that process millions of rows, transform data between layers, calculate running totals, identify duplicates, and rank records. This module covers the patterns you will use constantly from SK-02 onwards.

---

## Module 2 Glossary

**JOIN** — A SQL operation that combines rows from two tables based on a matching condition. The type of join (INNER, LEFT, RIGHT, FULL) determines what happens when there is no match.

**INNER JOIN** — Returns only rows where there is a match in BOTH tables. Non-matching rows are excluded.

**LEFT JOIN** — Returns ALL rows from the left table, and matching rows from the right table. Where there is no match, the right-side columns are NULL.

**GROUP BY** — Groups rows that share a value in one or more columns, so that aggregate functions (SUM, COUNT, AVG) operate per group rather than across all rows.

**Aggregate function** — A function that takes multiple rows and returns one value: SUM (total), COUNT (how many), AVG (average), MIN (smallest), MAX (largest).

**HAVING** — Filters groups AFTER the GROUP BY aggregation. WHERE filters individual rows; HAVING filters groups. "Show me only the customers who have spent more than $1,000 in total" requires HAVING.

**Subquery** — A query written inside another query. The inner query runs first, and its results are used by the outer query.

**CTE (Common Table Expression)** — A named temporary result set that you define at the top of a query using `WITH`. Makes complex queries much more readable than nested subqueries.

**Window function** — A function that performs a calculation across a set of related rows (a "window") without collapsing them into one row the way GROUP BY does. Allows you to calculate running totals, rankings, and row-to-row differences.

**PARTITION BY** — Used inside a window function to define the groups within which the calculation is performed. Like GROUP BY, but for window functions — it does not collapse the rows.

**ORDER BY (inside window functions)** — Defines the order in which the window function processes rows. Critical for ranking and running total calculations.

**ROW_NUMBER()** — Assigns a sequential integer to each row within a partition (1, 2, 3...). Two rows with equal values still get different numbers.

**RANK()** — Like ROW_NUMBER(), but rows with equal values get the same rank. The next rank skips numbers (1, 1, 3 — not 1, 1, 2).

**LAG()** — Returns the value of a column from a previous row in the partition. "What was the value in the previous month?" Use LAG.

**LEAD()** — Returns the value from a future row. "What will the value be next month?" Use LEAD.

**NULL** — The SQL representation of a missing or unknown value. NULL is not zero and not an empty string. NULL compared to anything equals NULL (not true or false), which is why `WHERE column = NULL` does not work — you must use `WHERE column IS NULL`.

**COALESCE** — A function that returns the first non-NULL value in a list. `COALESCE(amount, 0)` returns `amount` if it is not NULL, otherwise returns 0.

**CASE WHEN** — SQL's if-then-else logic. Lets you create conditional columns.

---

## Concept Explainer 2.1: JOINs — The Most Important SQL Concept

Understanding JOINs properly is the difference between writing queries that work and writing queries that look right but silently drop rows or create duplicate rows.

Think of two tables as two lists:
- **Orders table**: every purchase made (order_id, customer_id, amount)
- **Customers table**: every registered customer (customer_id, name, email)

The relationship is: every order has a `customer_id` that should match a `customer_id` in the customers table.

**INNER JOIN — the intersection**
Returns only rows where BOTH tables have a matching value. If an order has a `customer_id` that does not exist in the customers table, that order disappears from the result. This is the most common source of silent data loss in SQL queries.

```sql
-- Returns only orders from known customers
SELECT o.order_id, c.name, o.amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

**LEFT JOIN — keep everything from the left**
Returns ALL rows from the left table (orders), even if there is no match in the right table (customers). Where there is no match, the columns from the right table are NULL. Use this when you do not want to lose any rows from your primary table.

```sql
-- Returns ALL orders, with customer info where available
-- Orders with unknown customer_id get NULL for name and email
SELECT o.order_id, c.name, o.amount
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

**The duplicate rows trap**
If the customers table had multiple rows for the same `customer_id` (e.g. due to a data quality issue), a JOIN would produce multiple output rows for each order — your row count would explode and your totals would be wrong. Always check for duplicates in join keys before joining.

---

## Concept Explainer 2.2: GROUP BY — Collapsing Rows Into Summaries

GROUP BY takes many rows and collapses them into fewer rows, one per unique value of the grouping column. It only works when you also apply an aggregate function to the non-grouped columns.

**The analogy:** You have a pile of receipts. GROUP BY is the action of sorting the receipts into piles by month. The SUM is what you get when you add up each pile.

```sql
-- WRONG: Trying to show both individual order_id AND a group sum
-- This will cause an error in most databases
SELECT customer_id, order_id, SUM(amount)
FROM orders
GROUP BY customer_id;
-- Error: order_id must appear in GROUP BY or be used in an aggregate function

-- CORRECT: Either group both columns, or only select aggregated values
SELECT customer_id, COUNT(order_id) AS order_count, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

**HAVING vs WHERE**
- `WHERE` filters rows BEFORE the GROUP BY aggregation
- `HAVING` filters groups AFTER the aggregation

```sql
-- Find customers who have placed more than 5 orders AND spent more than $500
SELECT customer_id, COUNT(*) AS orders, SUM(amount) AS total
FROM orders
WHERE status = 'completed'        -- WHERE filters rows first
GROUP BY customer_id
HAVING COUNT(*) > 5               -- HAVING filters groups after aggregation
   AND SUM(amount) > 500;
```

---

## Concept Explainer 2.3: CTEs — Making Complex Queries Readable

A CTE (WITH clause) lets you name an intermediate query result and use it later in the same query. It does not save to the database. It just makes the query more readable.

**Without a CTE (hard to read):**
```sql
SELECT customer_id, total_spent, rank
FROM (
    SELECT customer_id,
           SUM(amount) AS total_spent,
           RANK() OVER (ORDER BY SUM(amount) DESC) AS rank
    FROM (
        SELECT customer_id, amount
        FROM orders
        WHERE status = 'completed'
          AND order_date >= '2026-01-01'
    ) completed_orders
    GROUP BY customer_id
) ranked
WHERE rank <= 10;
```

**With a CTE (readable — each step is named):**
```sql
WITH
-- Step 1: Filter to completed orders in 2026
completed_orders AS (
    SELECT customer_id, amount
    FROM orders
    WHERE status = 'completed'
      AND order_date >= '2026-01-01'
),

-- Step 2: Total spend per customer
customer_totals AS (
    SELECT customer_id, SUM(amount) AS total_spent
    FROM completed_orders
    GROUP BY customer_id
),

-- Step 3: Rank by total spend
ranked_customers AS (
    SELECT customer_id,
           total_spent,
           RANK() OVER (ORDER BY total_spent DESC) AS rank
    FROM customer_totals
)

-- Final: top 10
SELECT * FROM ranked_customers
WHERE rank <= 10;
```

Both queries produce identical results. The CTE version is readable by a human six months later. Data engineers write CTEs constantly. They are the foundation of dbt models.

---

## Concept Explainer 2.4: Window Functions — The Most Powerful SQL Tool

Window functions are the most powerful SQL feature most beginners have never seen. They let you perform calculations across related rows without collapsing the result.

**The key difference from GROUP BY:**
- GROUP BY collapses 10 rows into 1 summary row
- A window function adds a calculated column to EACH of the 10 rows

**ROW_NUMBER — numbering rows within a group**
```sql
-- Assign a number to each order per customer, newest first
-- This is the standard pattern for "get the most recent record per group"
SELECT
    order_id,
    customer_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id      -- restart numbering for each customer
        ORDER BY order_date DESC      -- number from newest to oldest
    ) AS order_rank
FROM orders;

-- Then wrap it to get only the most recent order per customer:
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
-- This pattern appears in almost every data pipeline you will build
```

**LAG — comparing to the previous row**
```sql
-- Calculate month-over-month revenue change
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS change
FROM monthly;
```

**Running total**
```sql
-- Cumulative revenue over time
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

---

## Worked Example 2.1: A Complete Data Engineering SQL Problem

This is the kind of query a data engineer writes to build a Gold analytics table from Silver source data.

**The task:** From an orders table, produce a customer summary showing each customer's: total orders, total spend, average order value, their most expensive single order, how long they have been a customer (in months), and their rank by total spend.

```sql
WITH
-- Step 1: Calculate metrics per customer
customer_metrics AS (
    SELECT
        customer_id,
        COUNT(order_id)                                AS total_orders,
        SUM(amount)                                    AS total_spend,
        AVG(amount)                                    AS avg_order_value,
        MAX(amount)                                    AS max_single_order,
        MIN(order_date)                                AS first_order_date,
        MAX(order_date)                                AS last_order_date
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),

-- Step 2: Add derived columns and rankings
enriched AS (
    SELECT
        customer_id,
        total_orders,
        ROUND(total_spend, 2)                          AS total_spend,
        ROUND(avg_order_value, 2)                      AS avg_order_value,
        ROUND(max_single_order, 2)                     AS max_single_order,
        first_order_date,
        last_order_date,
        -- How long have they been a customer? (in months)
        DATEDIFF('month', first_order_date, CURRENT_DATE) AS customer_age_months,
        -- Are they active? (ordered in the last 90 days)
        CASE
            WHEN last_order_date >= CURRENT_DATE - INTERVAL '90 days'
            THEN 'Active'
            ELSE 'Lapsed'
        END                                            AS customer_status,
        -- Rank by total spend (1 = highest spender)
        RANK() OVER (ORDER BY total_spend DESC)        AS spend_rank,
        -- Percentile (what percentage of customers spend less than this one?)
        ROUND(
            PERCENT_RANK() OVER (ORDER BY total_spend) * 100, 1
        )                                              AS spend_percentile
    FROM customer_metrics
)

SELECT * FROM enriched
ORDER BY spend_rank;
```

This is a real dbt model pattern. The CTEs name each logical step. The window functions add rankings without collapsing the rows. The CASE WHEN adds business logic. The ROUND function controls decimal precision.

---

## Worked Example 2.2: Finding and Handling Duplicates

Detecting and removing duplicate rows is a core data quality task. This is the SQL pattern every data engineer needs.

```sql
-- Step 1: Check if there are duplicates
SELECT
    order_id,
    COUNT(*) AS occurrences
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;
-- If this returns any rows, you have duplicates

-- Step 2: See the duplicate rows
SELECT *
FROM orders
WHERE order_id IN (
    SELECT order_id
    FROM orders
    GROUP BY order_id
    HAVING COUNT(*) > 1
)
ORDER BY order_id;

-- Step 3: Keep only the most recent version of each duplicate
-- (This is the standard deduplication pattern in data engineering)
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY order_id           -- the business key to deduplicate on
            ORDER BY updated_at DESC        -- keep the most recently updated row
        ) AS rn
    FROM orders
)
SELECT *
FROM ranked
WHERE rn = 1;   -- rn = 1 means this is the most recent row for this order_id
```

---

## Module 2 Self-Check

1. What is the difference between `WHERE` and `HAVING`? Write an example of a query that needs HAVING (not WHERE) to filter correctly.

2. You run this query and get 150,000 rows, but your orders table has only 100,000 rows. What has gone wrong and what do you check first?
   ```sql
   SELECT * FROM orders LEFT JOIN customers ON orders.customer_id = customers.customer_id;
   ```

3. Write the SQL to get the most recent order for each customer. (Hint: ROW_NUMBER + PARTITION BY.)

4. What does `LAG(amount) OVER (PARTITION BY customer_id ORDER BY order_date)` return for the very first order of each customer?

5. A CTE does not save data to the database. So why use one instead of just writing a longer query? Give two reasons.

---

## Module 2 Resources

### Read First (Free, Currently Live)
1. **Mode Analytics SQL Tutorial** (mode.com/sql-tutorial) — A free, browser-based SQL tutorial that goes well beyond the basics. Covers JOINs, aggregations, subqueries, window functions, and CTEs. You can run the queries directly in the browser with no setup. Start at "Advanced SQL" if you are comfortable with the basics.

2. **PostgreSQL Documentation: Window Functions** (postgresql.org/docs/current/tutorial-window.html) — The official tutorial for window functions. PostgreSQL's documentation is among the best-written technical documentation in existence. Read the window functions tutorial from start to finish.

3. **CTEs — learnsql.com** (learnsql.com/blog/what-is-common-table-expression) — A clear, illustrated explanation of CTEs with multiple practical examples. Free to read.

### Watch (Free, Currently Live)
1. **SQL Window Functions — Colt Steele** — Search "Colt Steele SQL window functions" on YouTube. Colt is one of the most popular technical educators on YouTube. His window functions video is clear, practical, and about 30 minutes.

2. **Advanced SQL Tutorial | Subqueries** — Search "Alex the Analyst SQL subqueries" on YouTube. Alex has a strong series on intermediate and advanced SQL topics with real data.

### Practice (Free, Interactive)
1. **SQLZoo** (sqlzoo.net) — Interactive SQL exercises in the browser. Do the "SUM and COUNT," "The JOIN operation," and "Using Null" sections. Estimated time: 3 hours.

2. **HackerRank SQL** (hackerrank.com/domains/sql) — Practice problems graded by difficulty. Do all "Easy" and "Medium" problems. Estimated time: 4 hours.

