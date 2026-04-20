# Module C: Dimensional Modeling & Star Schema

**Time:** 5-6 hours | **Focus:** Kimball methodology, fact/dimension design

---

## C.1 Dimensional Modeling Fundamentals

### Read

| Resource | Time | Why |
|----------|------|-----|
| [The Data Warehouse Toolkit (Kimball), Ch 1-3](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/) | 2 hrs | The bible of dimensional modeling |
| [Star Schema Introduction](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/) | 30 min | Free Kimball techniques |

---

## C.2 Core Concepts

### Fact Tables

- Contain **measurements** (metrics, facts)
- Foreign keys to dimensions
- Grain = "one row per ___"

```sql
-- Example: fact_sales
CREATE TABLE fact_sales (
    sale_key BIGSERIAL PRIMARY KEY,

    -- Dimension FKs
    date_key INT REFERENCES dim_date(date_key),
    product_key INT REFERENCES dim_product(product_key),
    store_key INT REFERENCES dim_store(store_key),

    -- Measures
    quantity INT,
    unit_price DECIMAL(10,2),
    revenue DECIMAL(12,2),
    discount_amount DECIMAL(10,2)
);
```

### Dimension Tables

- Contain **attributes** (descriptors)
- Surrogate keys (not natural keys)
- Often denormalized

```sql
-- Example: dim_product
CREATE TABLE dim_product (
    product_key SERIAL PRIMARY KEY,          -- Surrogate
    product_id VARCHAR(20),                   -- Natural key
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_cost DECIMAL(10,2)
);
```

---

## C.3 Star Schema Pattern

```
                    dim_date
                       │
                       │
dim_product ────── fact_sales ────── dim_store
                       │
                       │
                   dim_customer
```

**Rules:**
1. One fact table at center
2. Dimensions connect directly (no snowflake)
3. All relationships are many-to-one (fact → dim)
4. Surrogate keys everywhere

---

## C.4 Slowly Changing Dimensions

### Type 1: Overwrite

```sql
-- Simply update the row
UPDATE dim_customer
SET address = '456 New St'
WHERE customer_id = 'C001';

-- Use when: History doesn't matter
```

### Type 2: Add New Row

```sql
-- Current row
INSERT INTO dim_customer (customer_id, address, effective_start, effective_end, is_current)
VALUES ('C001', '456 New St', '2024-01-15', '9999-12-31', TRUE);

-- Close old row
UPDATE dim_customer
SET effective_end = '2024-01-14', is_current = FALSE
WHERE customer_id = 'C001' AND is_current = TRUE;

-- Use when: Need to track history
```

### SCD Type 2 Pattern

```sql
-- Step 1: Close existing current rows that have changes
UPDATE dim_customer dc
SET
    effective_end = CURRENT_DATE - 1,
    is_current = FALSE
FROM staging_customers sc
WHERE dc.customer_id = sc.customer_id
  AND dc.is_current = TRUE
  AND (dc.address IS DISTINCT FROM sc.address
       OR dc.segment IS DISTINCT FROM sc.segment);

-- Step 2: Insert new versions for changed records
INSERT INTO dim_customer (customer_id, address, segment, effective_start, effective_end, is_current)
SELECT customer_id, address, segment, CURRENT_DATE, '9999-12-31', TRUE
FROM staging_customers sc
WHERE EXISTS (
    SELECT 1 FROM dim_customer dc
    WHERE dc.customer_id = sc.customer_id
      AND dc.is_current = FALSE
      AND dc.effective_end = CURRENT_DATE - 1
);
```

---

## C.5 Date Dimension

```sql
-- Generate date dimension (PostgreSQL)
CREATE TABLE dim_date AS
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT AS date_key,
    d AS date,
    EXTRACT(YEAR FROM d) AS year,
    EXTRACT(QUARTER FROM d) AS quarter,
    EXTRACT(MONTH FROM d) AS month,
    TO_CHAR(d, 'Month') AS month_name,
    EXTRACT(DAY FROM d) AS day,
    TO_CHAR(d, 'Day') AS day_name,
    EXTRACT(DOW FROM d) AS day_of_week,
    CASE WHEN EXTRACT(DOW FROM d) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend
FROM GENERATE_SERIES('2020-01-01'::DATE, '2030-12-31'::DATE, '1 day') AS d;
```

---

## Checkpoint

Before moving to Module D:

- [ ] Design a star schema for a business scenario
- [ ] Write SCD Type 2 close-then-insert SQL
- [ ] Generate a date dimension with pure SQL
- [ ] Explain grain, surrogate keys, degenerate dimensions
