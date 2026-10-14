# Lab 05 — Loading the Dimensions

> **Goal:** Populate `dim_date`, the Type 1 dimensions (`dim_store`, `dim_promotion`) and the SCD Type 2 dimensions (`dim_customer`, `dim_product`) with idempotent SQL — the guided lab's Part 3 scripts.
> **Time:** 150–180 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 04
> **Guided lab coverage:** PART 3 (Steps 3.1–3.4) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Generate a full calendar dimension (fiscal year, holidays) with `GENERATE_SERIES`, `EXTRACT`, `TO_CHAR` and `CASE`.
- Write a Type 1 **UPSERT** with `INSERT … ON CONFLICT … DO UPDATE SET col = EXCLUDED.col`.
- **Denormalize** a snowflaked hierarchy (store → district → region, product → subcategory → category) into one dimension row.
- Stage source data in a **temporary table** and make scripts rerunnable with `DROP TABLE IF EXISTS`.
- Implement SCD Type 2 **close-then-insert** with `IS DISTINCT FROM` and `NOT EXISTS`, inside a transaction.
- Prove a load is **idempotent** by running it twice.

**Key terms**

| Term | Meaning |
|---|---|
| **Idempotent** | Running it twice leaves the same result as running it once. Nightly jobs get retried — every load must be idempotent. |
| **UPSERT** | "Insert, or update if it already exists" — PostgreSQL's `INSERT … ON CONFLICT`. |
| **Staging table** | A temporary copy of the source rows, already reshaped, that the load statements read from. |
| **Transaction** | `BEGIN … COMMIT`: all statements succeed together or none take effect. |

## Steps

### Step 1 — Populate dim_date (guided Step 3.1)

How the pieces work: `GENERATE_SERIES(start, end, '1 day')` produces one row per day; `TO_CHAR(d, 'YYYYMMDD')::INT` builds the key; `EXTRACT(ISODOW …)` gives 1 = Monday … 7 = Sunday; `TO_CHAR(d, 'Day')` pads names to 9 characters, so `TRIM` removes the spaces; the `CASE` blocks implement FreshMart's 1-February fiscal year; `ON CONFLICT (date_key) DO NOTHING` makes reruns harmless. Then three `UPDATE`s mark fixed-date US holidays.

Save as `C:\DataEngineering\sk02\sql\04_load_dim_date.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 04_load_dim_date.sql — populate dim_date (2020-01-01 to 2030-12-31)
-- Idempotent: ON CONFLICT skips dates that already exist.
-- ═══════════════════════════════════════════════════════════════

INSERT INTO dim_date
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT                 AS date_key,
    d::DATE                                     AS full_date,
    EXTRACT(ISODOW FROM d)::SMALLINT            AS day_of_week,
    TRIM(TO_CHAR(d, 'Day'))                     AS day_name,
    EXTRACT(DAY FROM d)::SMALLINT               AS day_of_month,
    EXTRACT(DOY FROM d)::SMALLINT               AS day_of_year,
    EXTRACT(WEEK FROM d)::SMALLINT              AS week_of_year,
    EXTRACT(MONTH FROM d)::SMALLINT             AS month_number,
    TRIM(TO_CHAR(d, 'Month'))                   AS month_name,
    TO_CHAR(d, 'Mon')                           AS month_short,
    EXTRACT(QUARTER FROM d)::SMALLINT           AS quarter_number,
    'Q' || EXTRACT(QUARTER FROM d)::TEXT        AS quarter_name,
    EXTRACT(YEAR FROM d)::INT                   AS year_number,
    TO_CHAR(d, 'YYYY-MM')                       AS year_month,
    -- FreshMart fiscal year starts Feb 1
    CASE
        WHEN EXTRACT(MONTH FROM d) >= 2 THEN EXTRACT(YEAR FROM d)::INT
        ELSE EXTRACT(YEAR FROM d)::INT - 1
    END                                         AS fiscal_year,
    CASE
        WHEN EXTRACT(MONTH FROM d) IN (2,3,4)   THEN 1
        WHEN EXTRACT(MONTH FROM d) IN (5,6,7)   THEN 2
        WHEN EXTRACT(MONTH FROM d) IN (8,9,10)  THEN 3
        ELSE 4
    END::SMALLINT                               AS fiscal_quarter,
    EXTRACT(ISODOW FROM d) IN (6,7)             AS is_weekend,
    FALSE                                       AS is_holiday,
    NULL::VARCHAR(50)                           AS holiday_name
FROM GENERATE_SERIES(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) AS d
ON CONFLICT (date_key) DO NOTHING;

-- Mark US public holidays (fixed-date only for simplicity)
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'New Year''s Day'
WHERE month_number = 1  AND day_of_month = 1;

UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Independence Day'
WHERE month_number = 7  AND day_of_month = 4;

UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Christmas Day'
WHERE month_number = 12 AND day_of_month = 25;

-- Verify
SELECT COUNT(*) AS days_generated, MIN(full_date), MAX(full_date) FROM dim_date;
```

Run it:

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/04_load_dim_date.sql
```

**Expected output:**

```
INSERT 0 4018
UPDATE 11
UPDATE 11
UPDATE 11
 days_generated |    min     |    max
----------------+------------+------------
           4018 | 2020-01-01 | 2030-12-31
(1 row)
```

Spot-check one day and the flags (interactive session):

```sql
SELECT date_key, day_name, day_of_week, week_of_year, month_name, quarter_name,
       year_month, fiscal_year, fiscal_quarter, is_weekend
FROM dim_date WHERE full_date = DATE '2025-01-15';

SELECT COUNT(*) FILTER (WHERE is_weekend) AS weekend_days,
       COUNT(*) FILTER (WHERE is_holiday) AS holidays
FROM dim_date;
```

**Expected output:**

```
 date_key | day_name  | day_of_week | week_of_year | month_name | quarter_name | year_month | fiscal_year | fiscal_quarter | is_weekend
----------+-----------+-------------+--------------+------------+--------------+------------+-------------+----------------+------------
 20250115 | Wednesday |           3 |            3 | January    | Q1           | 2025-01    |        2024 |              4 | f
(1 row)

 weekend_days | holidays
--------------+----------
         1148 |       33
(1 row)
```

Run the script a second time: `INSERT 0 0` (nothing new), the three `UPDATE 11` again, same totals — idempotent.

✅ **Checkpoint:** 4018 days; 15 Jan 2025 is in fiscal year 2024, Q4.

### Step 2 — Type 1 dimensions with UPSERT (guided Step 3.2)

The UPSERT pattern, on its own:

```sql
INSERT INTO target (natural_key, attr1, attr2)
SELECT natural_key, attr1, attr2 FROM source
ON CONFLICT (natural_key) DO UPDATE SET     -- needs a UNIQUE constraint on natural_key
    attr1 = EXCLUDED.attr1,                 -- EXCLUDED = the row you tried to insert
    attr2 = EXCLUDED.attr2;
```

The store load also **denormalizes**: it joins store → district → region (and the manager's name from `employees`) once, here, so no report ever has to. `LEFT JOIN employees` keeps a store even if it has no manager.

Save as `C:\DataEngineering\sk02\sql\05_load_type1_dims.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 05_load_type1_dims.sql — dim_store and dim_promotion (SCD Type 1)
-- Type 1 = overwrite. UPSERT makes every run idempotent.
-- ═══════════════════════════════════════════════════════════════

-- ── dim_store: denormalize store → district → region (+ manager) ──
INSERT INTO dim_store (
    store_id, store_name, store_type, district_name, region_name,
    city, state, country, postal_code, square_footage, opening_date, manager_name
)
SELECT
    s.store_id,
    s.store_name,
    s.store_type,
    sd.district_name,
    sr.region_name,                 -- Denormalize region into the store dim
    s.city,
    s.state,
    s.country,
    s.postal_code,
    s.square_footage,
    s.opening_date,
    CONCAT(e.first_name, ' ', e.last_name) AS manager_name
FROM source_oltp.stores s
JOIN source_oltp.store_districts sd ON s.district_id = sd.district_id
JOIN source_oltp.store_regions   sr ON sd.region_id   = sr.region_id
LEFT JOIN source_oltp.employees  e  ON s.manager_id   = e.employee_id
ON CONFLICT (store_id) DO UPDATE SET
    store_name     = EXCLUDED.store_name,
    store_type     = EXCLUDED.store_type,
    district_name  = EXCLUDED.district_name,
    region_name    = EXCLUDED.region_name,
    city           = EXCLUDED.city,
    state          = EXCLUDED.state,
    country        = EXCLUDED.country,
    postal_code    = EXCLUDED.postal_code,
    square_footage = EXCLUDED.square_footage,
    opening_date   = EXCLUDED.opening_date,
    manager_name   = EXCLUDED.manager_name;

-- ── dim_promotion: one row per source promotion ──────────────
-- (the -1 "No Promotion" row was inserted by the DDL and is never touched)
INSERT INTO dim_promotion (
    promotion_id, promotion_name, promotion_type, discount_percent,
    start_date, end_date, is_active
)
SELECT promotion_id, promotion_name, promotion_type, discount_percent,
       start_date, end_date, is_active
FROM source_oltp.promotions
ON CONFLICT (promotion_id) DO UPDATE SET
    promotion_name   = EXCLUDED.promotion_name,
    promotion_type   = EXCLUDED.promotion_type,
    discount_percent = EXCLUDED.discount_percent,
    start_date       = EXCLUDED.start_date,
    end_date         = EXCLUDED.end_date,
    is_active        = EXCLUDED.is_active;

-- Verify
SELECT (SELECT COUNT(*) FROM dim_store)     AS stores,
       (SELECT COUNT(*) FROM dim_promotion) AS promotions_incl_placeholder;
```

Run it (same `docker exec … -f /sql/05_load_type1_dims.sql` pattern).

**Expected output:**

```
INSERT 0 60
INSERT 0 40
 stores | promotions_incl_placeholder
--------+-----------------------------
     60 |                          41
(1 row)
```

Look at the result:

```sql
SELECT store_key, store_id, store_name, store_type, district_name, region_name, manager_name
FROM dim_store WHERE store_id IN (1, 60) ORDER BY store_id;
```

**Expected output:**

```
 store_key | store_id | store_name  | store_type  |   district_name   | region_name | manager_name
-----------+----------+-------------+-------------+-------------------+-------------+--------------
         1 |        1 | New York #1 | Supermarket | New York District | Northeast   | Mary Davis
        60 |       60 | Denver #60  | Online      | Denver District   | West        | James Smith
(2 rows)
```

Rerun the script: again `INSERT 0 60` / `INSERT 0 40` — PostgreSQL reports *upserted* rows — and the totals stay 60 and 41.

✅ **Checkpoint:** one row per store with region and manager names filled in; reruns never add rows.

### Step 3 — The building blocks of SCD Type 2

Try these three ideas in an interactive session before reading the SCD2 script:

```sql
-- 1. NULL-safe comparison: <> misses changes to/from NULL
SELECT 'a' <> NULL                AS not_equal,
       'a' IS DISTINCT FROM NULL  AS is_distinct,
       NULL IS DISTINCT FROM NULL AS both_null;

-- 2. NOT EXISTS: "rows with no matching row over there"
SELECT COUNT(*) AS customers_never_ordered
FROM source_oltp.customers c
WHERE NOT EXISTS (SELECT 1 FROM source_oltp.orders o WHERE o.customer_id = c.customer_id);

-- 3. A temporary table lives only for this session
CREATE TEMPORARY TABLE tmp_demo AS SELECT 1 AS x;
DROP TABLE IF EXISTS tmp_demo;
DROP TABLE IF EXISTS tmp_demo;
```

**Expected output:**

```
 not_equal | is_distinct | both_null
-----------+-------------+-----------
           | t           | f
(1 row)

 customers_never_ordered
-------------------------
                     147
(1 row)

SELECT 1
DROP TABLE
NOTICE:  table "tmp_demo" does not exist, skipping
DROP TABLE
```

With `<>`, a customer whose email goes from NULL to a value is compared as *unknown*, which `WHERE` treats as *not changed* — the change would be silently lost. `DROP TABLE IF EXISTS` before `CREATE TEMPORARY TABLE` is what lets the SCD2 scripts rerun in the same session.

✅ **Checkpoint:** `not_equal` is empty (NULL), `is_distinct` is `t`.

### Step 4 — SCD Type 2 for dim_customer (guided Step 3.3)

Read the script in this order:

1. **Stage** — one row per customer, with segment name and primary address denormalized from three source tables.
2. **Step A (close)** — for current rows whose *tracked* attributes changed, set `effective_end = CURRENT_DATE - 1` and `is_current = FALSE`.
3. **Step B (insert)** — every customer **without** a current row (brand new, or just closed) gets a new current version. The first version ever is valid from `1900-01-01` so that *all* historical orders can find it; a changed customer's new version starts today. `ORDER BY stg.customer_id` assigns surrogate keys in a predictable order.
4. **Step C (Type 1)** — untracked attributes (name, phone) are simply overwritten.
5. All inside `BEGIN … COMMIT`, then a check that no customer has two current rows.

> **Why `1900-01-01`?** If the first version started on the load date, an order from 2024 would fall *before* every customer version, Lab 06's date-range lookup would find nothing, and the inner join would silently drop every historical sale.

Save as `C:\DataEngineering\sk02\sql\06_scd2_dim_customer.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 06_scd2_dim_customer.sql — SCD Type 2 load for dim_customer
-- Run daily during the nightly ETL window. Idempotent: a rerun on
-- unchanged source data closes nothing and inserts nothing.
-- ═══════════════════════════════════════════════════════════════

BEGIN;

-- Stage the current state from the OLTP source.
-- DROP first so the script can be rerun in the same psql session.
DROP TABLE IF EXISTS stg_customer;
CREATE TEMPORARY TABLE stg_customer AS
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.phone,
    cs.segment_name                         AS segment,
    ca.city,
    ca.state,
    ca.country,
    ca.postal_code,
    c.registration_date,
    c.updated_at                            AS source_updated_at
FROM source_oltp.customers          c
LEFT JOIN source_oltp.customer_segments cs
    ON c.segment_id   = cs.segment_id
LEFT JOIN source_oltp.customer_addresses ca
    ON c.customer_id  = ca.customer_id
    AND ca.is_primary = TRUE;

-- STEP A: Close current rows whose tracked (Type 2) attributes changed.
-- effective_end is inclusive, so the old version ends yesterday.
UPDATE dim_customer dc
SET
    effective_end = CURRENT_DATE - 1,
    is_current    = FALSE
FROM stg_customer stg
WHERE dc.customer_id = stg.customer_id
  AND dc.is_current  = TRUE
  AND (
      -- IS DISTINCT FROM handles NULLs correctly (NULL vs value = changed)
      dc.segment      IS DISTINCT FROM stg.segment
      OR dc.city      IS DISTINCT FROM stg.city
      OR dc.state     IS DISTINCT FROM stg.state
      OR dc.email     IS DISTINCT FROM stg.email
  );

-- STEP B: Insert new versions for changed records AND brand-new customers.
-- A customer needs a new version if no is_current=TRUE row exists.
INSERT INTO dim_customer (
    customer_id, first_name, last_name, email, phone,
    segment, city, state, country, postal_code,
    registration_date, effective_start, effective_end,
    is_current, source_updated_at
)
SELECT
    stg.customer_id,
    stg.first_name,
    stg.last_name,
    stg.email,
    stg.phone,
    stg.segment,
    stg.city,
    stg.state,
    stg.country,
    stg.postal_code,
    stg.registration_date,
    -- First version ever seen for this customer → valid since "the beginning of time",
    -- so ALL historical orders find it. A changed customer's new version starts today.
    CASE WHEN EXISTS (SELECT 1 FROM dim_customer h WHERE h.customer_id = stg.customer_id)
         THEN CURRENT_DATE
         ELSE DATE '1900-01-01'
    END                 AS effective_start,
    '9999-12-31'::DATE  AS effective_end,
    TRUE                AS is_current,
    stg.source_updated_at
FROM stg_customer stg
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer dc
    WHERE dc.customer_id = stg.customer_id
      AND dc.is_current  = TRUE
)
ORDER BY stg.customer_id;   -- deterministic surrogate keys: assigned in natural-key order

-- STEP C: Type 1 attributes (name, phone) — overwrite on every version
UPDATE dim_customer dc
SET first_name = stg.first_name,
    last_name  = stg.last_name,
    phone      = stg.phone
FROM stg_customer stg
WHERE dc.customer_id = stg.customer_id
  AND (dc.first_name IS DISTINCT FROM stg.first_name
       OR dc.last_name IS DISTINCT FROM stg.last_name
       OR dc.phone     IS DISTINCT FROM stg.phone);

COMMIT;

-- Verify: every customer_id has exactly ONE is_current=TRUE row
SELECT customer_id, COUNT(*) AS current_row_count
FROM dim_customer
WHERE is_current = TRUE
GROUP BY customer_id
HAVING COUNT(*) > 1;
-- Expected: 0 rows

SELECT COUNT(*) AS total_rows,
       COUNT(*) FILTER (WHERE is_current) AS current_rows
FROM dim_customer;
```

Run it.

**Expected output (first run):**

```
BEGIN
psql:/sql/06_scd2_dim_customer.sql:11: NOTICE:  table "stg_customer" does not exist, skipping
DROP TABLE
SELECT 20000
UPDATE 0
INSERT 0 20000
UPDATE 0
COMMIT
 customer_id | current_row_count
-------------+-------------------
(0 rows)

 total_rows | current_rows
------------+--------------
      20000 |        20000
(1 row)
```

**Run it again.** Expected: the same output except `INSERT 0 0` — nothing changed in the source, so nothing was closed or inserted. (The NOTICE appears again because each `docker exec` is a new session, so the temp table is gone.)

Look at one customer:

```sql
SELECT customer_key, customer_id, segment, city, state, effective_start, effective_end, is_current
FROM dim_customer WHERE customer_id = 42;
```

**Expected output:**

```
 customer_key | customer_id | segment  |  city   | state | effective_start | effective_end | is_current
--------------+-------------+----------+---------+-------+-----------------+---------------+------------
           42 |          42 | Standard | Atlanta | GA    | 1900-01-01      | 9999-12-31    | t
(1 row)
```

✅ **Checkpoint:** 20000 rows, all current, second run inserts 0.

### Step 5 — SCD Type 2 for dim_product (guided Step 3.4)

Same pattern. The staging query denormalizes product → subcategory → category and supplier. Tracked (Type 2): subcategory, category, unit_cost, list_price. Overwritten (Type 1): name, brand, supplier, weight, active flag.

Save as `C:\DataEngineering\sk02\sql\07_scd2_dim_product.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 07_scd2_dim_product.sql — SCD Type 2 load for dim_product
-- Same close-then-insert pattern as dim_customer.
-- Type 2 (tracked): subcategory, category, unit_cost, list_price.
-- Type 1 (overwrite): name, brand, supplier, weight, is_active.
-- ═══════════════════════════════════════════════════════════════

BEGIN;

-- Stage: denormalize product → subcategory → category (+ supplier)
DROP TABLE IF EXISTS stg_product;
CREATE TEMPORARY TABLE stg_product AS
SELECT
    p.product_id,
    p.product_name,
    p.brand_name,
    ps.subcategory_name,
    pc.category_name,
    sp.supplier_name,
    p.unit_cost,
    p.list_price,
    p.weight_kg,
    p.is_active
FROM source_oltp.products               p
JOIN source_oltp.product_subcats        ps ON p.subcategory_id = ps.subcategory_id
JOIN source_oltp.product_categories     pc ON ps.category_id   = pc.category_id
LEFT JOIN source_oltp.suppliers         sp ON p.supplier_id    = sp.supplier_id;

-- STEP A: Close current versions whose tracked attributes changed
UPDATE dim_product dp
SET effective_end = CURRENT_DATE - 1,
    is_current    = FALSE
FROM stg_product stg
WHERE dp.product_id = stg.product_id
  AND dp.is_current = TRUE
  AND (
      dp.subcategory_name IS DISTINCT FROM stg.subcategory_name
      OR dp.category_name IS DISTINCT FROM stg.category_name
      OR dp.unit_cost     IS DISTINCT FROM stg.unit_cost
      OR dp.list_price    IS DISTINCT FROM stg.list_price
  );

-- STEP B: Insert new versions (changed + brand-new products)
INSERT INTO dim_product (
    product_id, product_name, brand_name, subcategory_name, category_name,
    supplier_name, unit_cost, list_price, weight_kg, is_active,
    effective_start, effective_end, is_current
)
SELECT
    stg.product_id, stg.product_name, stg.brand_name, stg.subcategory_name,
    stg.category_name, stg.supplier_name, stg.unit_cost, stg.list_price,
    stg.weight_kg, stg.is_active,
    CASE WHEN EXISTS (SELECT 1 FROM dim_product h WHERE h.product_id = stg.product_id)
         THEN CURRENT_DATE
         ELSE DATE '1900-01-01'
    END,
    '9999-12-31'::DATE,
    TRUE
FROM stg_product stg
WHERE NOT EXISTS (
    SELECT 1 FROM dim_product dp
    WHERE dp.product_id = stg.product_id
      AND dp.is_current = TRUE
)
ORDER BY stg.product_id;   -- deterministic surrogate keys: assigned in natural-key order

-- STEP C: Type 1 attributes — overwrite on every version
UPDATE dim_product dp
SET product_name  = stg.product_name,
    brand_name    = stg.brand_name,
    supplier_name = stg.supplier_name,
    weight_kg     = stg.weight_kg,
    is_active     = stg.is_active
FROM stg_product stg
WHERE dp.product_id = stg.product_id
  AND (dp.product_name     IS DISTINCT FROM stg.product_name
       OR dp.brand_name    IS DISTINCT FROM stg.brand_name
       OR dp.supplier_name IS DISTINCT FROM stg.supplier_name
       OR dp.weight_kg     IS DISTINCT FROM stg.weight_kg
       OR dp.is_active     IS DISTINCT FROM stg.is_active);

COMMIT;

-- Verify
SELECT product_id, COUNT(*) FROM dim_product WHERE is_current GROUP BY product_id HAVING COUNT(*) > 1;
-- Expected: 0 rows
SELECT COUNT(*) AS total_rows,
       COUNT(*) FILTER (WHERE is_current) AS current_rows
FROM dim_product;
```

Run it.

**Expected output (first run):**

```
BEGIN
psql:/sql/07_scd2_dim_product.sql:11: NOTICE:  table "stg_product" does not exist, skipping
DROP TABLE
SELECT 1200
UPDATE 0
INSERT 0 1200
UPDATE 0
COMMIT
 product_id | count
------------+-------
(0 rows)

 total_rows | current_rows
------------+--------------
       1200 |         1200
(1 row)
```

```sql
SELECT product_key, product_name, subcategory_name, category_name, supplier_name,
       unit_cost, list_price, effective_start, is_current
FROM dim_product WHERE product_id = 7;
```

**Expected output:**

```
 product_key |  product_name   | subcategory_name | category_name | supplier_name | unit_cost | list_price | effective_start | is_current
-------------+-----------------+------------------+---------------+---------------+-----------+------------+-----------------+------------
           7 | Summit Yogurt 7 | Yogurt           | Dairy         | Supplier 10   |     22.06 |      34.88 | 1900-01-01      | t
(1 row)
```

You'll watch real SCD2 changes happen in Lab 06, after the fact table is loaded.

✅ **Checkpoint:** 1200 products, all current.

### Step 6 — Check all dimensions at once

```sql
SELECT 'dim_date' AS dimension, COUNT(*) AS rows FROM dim_date
UNION ALL SELECT 'dim_store',     COUNT(*) FROM dim_store
UNION ALL SELECT 'dim_promotion', COUNT(*) FROM dim_promotion
UNION ALL SELECT 'dim_customer',  COUNT(*) FROM dim_customer
UNION ALL SELECT 'dim_product',   COUNT(*) FROM dim_product;
```

**Expected output:**

```
   dimension   | rows
---------------+-------
 dim_date      |  4018
 dim_store     |    60
 dim_promotion |    41
 dim_customer  | 20000
 dim_product   |  1200
(5 rows)
```

`UNION ALL` stacks the results of several queries (all must have the same columns).

✅ **Checkpoint:** all five counts match. You have now done the guided lab's Part 3.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `there is no unique or exclusion constraint matching the ON CONFLICT specification` | The conflict column isn't `UNIQUE` | Use the Lab 04 DDL (`dim_store.store_id` and `dim_promotion.promotion_id` are `UNIQUE`). |
| `relation "stg_customer" already exists` | Your copy lacks `DROP TABLE IF EXISTS` | Use the script as given. |
| `duplicate key value violates unique constraint "uq_dim_customer_one_current"` | Step B ran without Step A, or the script was edited | Rerun `03_dw_ddl.sql`, then Labs 05 steps in order. |
| Day names compare unequal (`'Monday   '`) | `TRIM` missing around `TO_CHAR(d,'Day')` | Use the script as given. |
| `UPDATE` / `INSERT` counts > 0 on a rerun with no source changes | Source data changed, or the change detection compares untracked columns | Rerun the seed (Lab 01) or check Step A's column list. |
| Dimension counts are 0 after rerunning `03_dw_ddl.sql` | The DDL drops and recreates the warehouse | Rerun Steps 1, 2, 4 and 5. |

## Key takeaways

- `dim_date` is generated once with `GENERATE_SERIES` and enriched with fiscal and holiday attributes.
- Type 1 = UPSERT (`ON CONFLICT … DO UPDATE … EXCLUDED`); denormalize hierarchies at load time.
- Type 2 = stage → close changed (`IS DISTINCT FROM`) → insert missing current (`NOT EXISTS`) → overwrite Type 1 columns, in one transaction; first version starts `1900-01-01`.
- Every script is rerunnable and idempotent — prove it by running it twice.

## Next

[Lab 06 — Loading the Fact Table](Lab_06_Loading_the_Fact_Table.md): incremental, idempotent fact loading with SCD-aware key lookups — then watch SCD2 react to source changes.
