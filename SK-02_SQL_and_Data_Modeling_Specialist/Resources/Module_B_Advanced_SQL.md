# Module B: Advanced SQL (Window Functions & CTEs)

**Time:** 5-6 hours | **Focus:** Window functions, CTEs, subqueries

---

## B.1 Window Functions

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Window Functions - PostgreSQL Docs](https://www.postgresql.org/docs/current/tutorial-window.html) | 45 min | Official reference |
| [Window Functions - Mode Analytics](https://mode.com/sql-tutorial/sql-window-functions/) | 30 min | Interactive examples |

---

## B.2 Window Function Syntax

```sql
function_name() OVER (
    PARTITION BY col1, col2    -- Group without collapsing rows
    ORDER BY col3              -- Order within partition
    ROWS BETWEEN ...           -- Frame specification
)
```

### Essential Functions

```sql
-- RANKING
ROW_NUMBER() OVER (ORDER BY sales DESC)         -- 1, 2, 3, 4
RANK() OVER (ORDER BY sales DESC)               -- 1, 2, 2, 4 (gaps)
DENSE_RANK() OVER (ORDER BY sales DESC)         -- 1, 2, 2, 3 (no gaps)

-- LAG / LEAD
LAG(sales, 1) OVER (ORDER BY month)             -- Previous row
LEAD(sales, 1) OVER (ORDER BY month)            -- Next row
LAG(sales, 1, 0) OVER (ORDER BY month)          -- With default

-- RUNNING TOTALS
SUM(sales) OVER (ORDER BY month)                -- Cumulative sum
SUM(sales) OVER (
    PARTITION BY region ORDER BY month
)                                               -- Cumulative per region

-- MOVING AVERAGES
AVG(sales) OVER (
    ORDER BY month
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)                                               -- 3-month moving avg
```

---

## B.3 Common Patterns

```sql
-- YoY Comparison
SELECT
    month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS revenue_ly,
    revenue - LAG(revenue, 12) OVER (ORDER BY month) AS yoy_change
FROM monthly_sales;

-- Top N per Group
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM products
)
SELECT * FROM ranked WHERE rn <= 3;

-- Running Total with Reset
SELECT
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY EXTRACT(YEAR FROM order_date)
        ORDER BY order_date
    ) AS ytd_total
FROM orders;
```

---

## B.4 CTEs (Common Table Expressions)

```sql
-- Basic CTE
WITH monthly_totals AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total
    FROM orders
    GROUP BY 1
)
SELECT * FROM monthly_totals WHERE total > 10000;

-- Multiple CTEs
WITH
    raw_data AS (...),
    cleaned AS (SELECT * FROM raw_data WHERE ...),
    aggregated AS (SELECT ... FROM cleaned GROUP BY ...)
SELECT * FROM aggregated;

-- Recursive CTE (hierarchies)
WITH RECURSIVE org_chart AS (
    -- Base: top-level
    SELECT id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: each level down
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart;
```

---

## Checkpoint

Before moving to Module C:

- [ ] Write queries with ROW_NUMBER, RANK, LAG/LEAD
- [ ] Create running totals and moving averages
- [ ] Use CTEs to organize complex queries
- [ ] Implement "Top N per group" pattern
