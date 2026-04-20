# Module A: SQL Foundations & Core Querying

**Time:** 4-5 hours | **Focus:** SELECT, JOINs, NULL handling, execution order

---

## A.1 Relational Databases & Core SELECT

### Read

| Resource | Time | Why |
|----------|------|-----|
| [PostgreSQL Tutorial, Ch 1-5](https://www.postgresql.org/docs/current/tutorial.html) | 2 hrs | Clearest relational DB intro |
| [SQL Execution Order - Julia Evans](https://jvns.ca/blog/2019/10/03/sql-queries-don-t-start-with-select/) | 15 min | Prevents 90% of SQL confusion |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [SQL for Beginners - freeCodeCamp](https://www.youtube.com/watch?v=HXV3zeQKqGY) | 4 hrs | Comprehensive free tutorial |

---

## A.2 Execution Order

```
Written order:                Logical execution order:
─────────────                ──────────────────────────
SELECT col, AGG(col)         1. FROM + JOINs
FROM table                   2. WHERE
JOIN other ON ...            3. GROUP BY
WHERE condition              4. Aggregate functions
GROUP BY col                 5. HAVING
HAVING condition             6. SELECT (aliases created here)
ORDER BY col                 7. DISTINCT
LIMIT n                      8. ORDER BY
                             9. LIMIT / OFFSET

KEY IMPLICATIONS:
- Aliases in SELECT can't be used in WHERE or GROUP BY
- Aliases CAN be used in ORDER BY
- Window functions execute after WHERE, GROUP BY, HAVING
```

---

## A.3 NULL Handling

```sql
-- Comparisons: always use IS NULL / IS NOT NULL
WHERE col = NULL           -- WRONG: always returns FALSE
WHERE col IS NULL          -- CORRECT

-- Arithmetic: NULL propagates
SELECT 100 + NULL          -- Returns NULL
SELECT COALESCE(col, 0)    -- Replace NULL with default

-- Aggregates: ignore NULLs (except COUNT(*))
SELECT COUNT(col)          -- Non-NULL values only
SELECT COUNT(*)            -- All rows
SELECT AVG(col)            -- Denominator = non-NULL count

-- ORDER BY: control NULL position
ORDER BY col ASC NULLS LAST
ORDER BY col DESC NULLS FIRST
```

---

## A.4 JOINs Quick Reference

```sql
-- INNER: Only matching rows
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- LEFT: All from left, matched from right
SELECT * FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
-- Right side is NULL when no match

-- FULL OUTER: All from both
SELECT * FROM orders o
FULL OUTER JOIN customers c ON o.customer_id = c.id;

-- CROSS: Every combination (caution: n × m rows)
SELECT * FROM sizes CROSS JOIN colors;
```

### JOIN Danger: Fanout

```sql
-- If customers has duplicate ids, this explodes row count
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Always check: SELECT COUNT(*) before and after JOIN
```

---

## Checkpoint

Before moving to Module B:

- [ ] Write SELECT with WHERE, GROUP BY, HAVING
- [ ] Explain why aliases can't be used in WHERE
- [ ] Handle NULLs correctly in aggregations
- [ ] Choose the right JOIN type for the use case
