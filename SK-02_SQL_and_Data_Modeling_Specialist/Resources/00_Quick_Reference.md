# SK-02 Quick Reference

One-page SQL and dimensional modeling essentials.

---

## SQL Execution Order

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

---

## Window Functions

```sql
-- Ranking
ROW_NUMBER() OVER (ORDER BY sales DESC)
RANK() OVER (PARTITION BY region ORDER BY sales DESC)

-- Previous/Next
LAG(sales, 1) OVER (ORDER BY month)
LEAD(sales, 1) OVER (ORDER BY month)

-- Running totals
SUM(sales) OVER (ORDER BY month)

-- Moving average
AVG(sales) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

---

## Dimensional Model Checklist

```
□ Define grain: "One row per ___"
□ Identify measures → Fact table
□ Identify attributes → Dimension tables
□ Surrogate keys (SERIAL) on all dimensions
□ SCD Type 1 or 2 decision per dimension
□ Foreign key indexes on fact table
```

---

## SCD Type 2 Pattern

```sql
-- Step 1: Close changed rows
UPDATE dim_customer SET
    effective_end = CURRENT_DATE - 1,
    is_current = FALSE
WHERE customer_id = :id AND is_current = TRUE
  AND address IS DISTINCT FROM :new_address;

-- Step 2: Insert new version
INSERT INTO dim_customer (customer_id, address, effective_start, effective_end, is_current)
VALUES (:id, :new_address, CURRENT_DATE, '9999-12-31', TRUE);
```

---

## Index Strategy

| Column | Index? |
|--------|--------|
| Primary key | Auto |
| Foreign key | Yes |
| WHERE clause | Yes |
| Low cardinality | No |

---

## Date Dimension (PostgreSQL)

```sql
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT AS date_key,
    d AS date,
    EXTRACT(YEAR FROM d) AS year,
    EXTRACT(MONTH FROM d) AS month
FROM GENERATE_SERIES('2020-01-01'::DATE, '2030-12-31'::DATE, '1 day') AS d;
```
