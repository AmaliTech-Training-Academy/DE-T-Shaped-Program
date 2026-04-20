# SK-02 END-TO-END GUIDED LAB: Designing & Building a Retail Analytics Data Warehouse for FreshMart

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **data engineer at FreshMart**, a national retail chain with 200 stores and 15 million annual transactions. FreshMart runs its operations on a PostgreSQL OLTP database — a tightly normalized 3NF schema built for transactional integrity, not analytics.

The BI team has reached a breaking point:

- The **daily sales report takes 45 minutes** to run, requiring 8 joins across 30+ tables
- **Month-end financial reconciliation** takes 3 days of manual SQL work by 2 overloaded analysts
- Reporting queries cause **lock contention** on the OLTP database, slowing down point-of-sale transactions during peak retail hours
- The OLTP system only retains **90 days of data** — the business needs 5 years for trend analysis
- Business users cannot write their own queries — the schema is too complex for self-service

The CTO has mandated a dedicated analytics data warehouse. Your job: design the dimensional model, write the DDL, build the ETL SQL, implement SCD Type 2, and prove performance improvement with analytical queries.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why a retail DW?** | Retail is the most universally understood domain. Every concept you build here — grain, conformed dimensions, SCD, fact table design — transfers directly to healthcare, finance, logistics, and HR data warehouses. |
| **Why start with the OLTP problem?** | The business case for a DW always starts with pain on the source system. Understanding what makes the OLTP schema wrong for analytics is what makes your DW design decisions defensible. |
| **When would you build this on a project?** | Any time a client is running reporting queries on their operational database, has complex multi-join reports, needs historical tracking, or wants self-service BI. This pattern repeats across every industry. |
| **How is this different from just writing better SQL on the OLTP?** | Better indexes on the OLTP help at the margins — but 8 joins across 30+ tables cannot be made fast enough for self-service BI. The dimensional model solves the structural problem, not just the surface symptom. |

---

## WHAT YOU WILL BUILD

```
FreshMart Analytics Data Warehouse:
│
├── 1. SOURCE ANALYSIS
│   └── OLTP schema inspection + slow query diagnosis
│
├── 2. DIMENSIONAL MODEL (Star Schema)
│   ├── dim_date          (2020–2030, fiscal calendar, holidays)
│   ├── dim_customer      (SCD Type 2 — tracks segment & address changes)
│   ├── dim_product       (SCD Type 2 — tracks category & price changes)
│   ├── dim_store         (SCD Type 1 — overwrite)
│   ├── dim_promotion     (slowly changing, Type 1)
│   └── fact_sales        (grain: one row per order line item)
│
├── 3. ETL SQL
│   ├── dim_date population (GENERATE_SERIES)
│   ├── SCD Type 2 MERGE for dim_customer and dim_product
│   ├── Incremental fact_sales load (high-water mark)
│   └── Surrogate key lookups with SCD-aware date-range joins
│
├── 4. ANALYTICAL QUERIES
│   ├── Monthly sales by region/category (the 45-min query, now < 2s)
│   ├── Year-over-year growth by store
│   ├── Customer cohort retention (window functions)
│   ├── Running totals and 7/30-day moving averages
│   └── Pareto ABC product classification
│
└── 5. PERFORMANCE OPTIMIZATION
    ├── Composite covering indexes
    ├── Materialized view for the BI dashboard query
    └── Table partitioning strategy
```

### Time Estimate: 8–10 hours

---

## PREREQUISITES

- PostgreSQL 14+ (locally via Docker, or any cloud PostgreSQL instance)
- A SQL client: psql, DBeaver, pgAdmin, or VS Code SQL extension
- Optional: Access to a cloud data warehouse (Redshift, Snowflake, BigQuery) to run the platform-variant notes

---

## PART 1: UNDERSTAND THE PROBLEM — SOURCE ANALYSIS (45 min)

### Step 1.1: Inspect the OLTP Schema

**WHY:** Before designing a warehouse, you must understand the source. Knowing the OLTP schema tells you which normalizations to undo, which surrogate keys exist, and which tables hold the data you need.

**HOW:**

```sql
-- ═══════════════════════════════════════════════════════════════
-- PART 1: OLTP SOURCE ANALYSIS
-- FreshMart operational PostgreSQL database
-- ═══════════════════════════════════════════════════════════════

-- Step 1.1: Understand data volumes
SELECT
    relname          AS table_name,
    n_live_tup       AS estimated_row_count,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_live_tup DESC;

-- Expected results in the FreshMart OLTP:
-- order_items:            ~15,000,000
-- orders:                  ~4,200,000
-- customers:                 ~850,000
-- products:                   ~12,000 (active + discontinued)
-- stores:                        ~200
-- promotions:                  ~3,500
-- employees:                   ~8,000
```

### Step 1.2: Run the Current "Simple" Sales Report and Diagnose It

**WHY:** EXPLAIN ANALYZE is your diagnostic tool. Before writing a single line of DW code, you need to understand exactly why the current query is slow. This gives you the before picture for your performance proof.

```sql
-- Step 1.2: Diagnose the slow query
-- This is the query that takes 45 minutes on the OLTP database
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    s.store_name,
    sr.region_name,
    pc.category_name,
    p.product_name,
    DATE_TRUNC('month', o.order_date) AS sales_month,
    COUNT(DISTINCT o.order_id)                         AS order_count,
    SUM(oi.quantity)                                   AS units_sold,
    SUM(oi.quantity * oi.unit_price)                   AS gross_revenue,
    SUM(oi.discount_amount)                            AS total_discounts,
    SUM(oi.quantity * oi.unit_price - oi.discount_amount) AS net_revenue
FROM orders o
JOIN order_items       oi  ON o.order_id       = oi.order_id
JOIN products          p   ON oi.product_id    = p.product_id
JOIN product_subcats   ps  ON p.subcategory_id = ps.subcategory_id
JOIN product_categories pc ON ps.category_id  = pc.category_id
JOIN stores            s   ON o.store_id       = s.store_id
JOIN store_districts   sd  ON s.district_id    = sd.district_id
JOIN store_regions     sr  ON sd.region_id     = sr.region_id
WHERE o.order_date >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY
    s.store_name, sr.region_name, pc.category_name,
    p.product_name, DATE_TRUNC('month', o.order_date)
ORDER BY sr.region_name, s.store_name, sales_month;

-- WHAT YOU WILL SEE IN THE PLAN:
-- Seq Scan on order_items (15M rows) — no index on order_date or product_id
-- Seq Scan on orders (4.2M rows)
-- 8 Hash Joins materializing large intermediate result sets
-- GROUP BY on text columns (slow — no surrogate keys)
-- Running on OLTP = competing with live transactions for I/O

-- WHY THIS FAILS:
-- 1. 8 JOINs across 30+ normalized tables
-- 2. Missing indexes on FK columns
-- 3. Full table scans on every access
-- 4. GROUP BY forces a sort on millions of rows of string data
-- 5. OLTP database is not the right tool for this workload
```

---

## PART 2: DESIGN THE DIMENSIONAL MODEL (60 min)

### Step 2.1: Declare the Grain and Design Decisions

**WHY:** Grain is the single most important decision in DW design. Get it wrong and every query either breaks or produces wrong numbers. The grain declaration should be written down before a single table is created.

```sql
-- ═══════════════════════════════════════════════════════════════
-- DESIGN DECISIONS — DOCUMENT THESE BEFORE WRITING DDL
-- ═══════════════════════════════════════════════════════════════

-- GRAIN: One row per order line item
-- (one product within one order at one store on one date)
-- WHY: Most granular natural unit. Can aggregate to any higher level:
--   store total = SUM over all products for that store
--   daily total = SUM over all stores and products for that date
--   customer lifetime = SUM over all orders for that customer

-- SCHEMA TYPE: Star (not snowflake)
-- WHY: Fewer joins for BI tools, better for Power BI/Tableau DAX
-- Trade-off: Some dimension redundancy (category_name stored in dim_product)

-- SCD STRATEGY:
--   dim_customer: Type 2 (track segment & address changes for historical accuracy)
--   dim_product:  Type 2 (track category & price changes for margin analysis)
--   dim_store:    Type 1 (overwrite — hierarchy rarely changes; no analytical need for history)
--   dim_promotion: Type 1 (overwrite — promotions don't retroactively change)

-- CONFORMED DIMENSIONS:
--   dim_date is the same table used by all future fact tables in this warehouse
--   dim_store is the same table used by fact_sales and any future fact_inventory
--   This enables cross-fact-table analysis without ETL duplication
```

### Step 2.2: Create All Dimension Tables

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 2: CREATE DIMENSION TABLES
-- ═══════════════════════════════════════════════════════════════

-- ── DATE DIMENSION ────────────────────────────────────────────
-- WHY: Every DW requires a date dimension. It enables time
-- intelligence (fiscal calendar, holidays, week-of-year) that
-- cannot be derived reliably from raw TIMESTAMP values.
-- The date_key in YYYYMMDD integer format sorts correctly,
-- is human-readable, and joins faster than a DATE column.

CREATE TABLE dim_date (
    date_key        INT         PRIMARY KEY,       -- YYYYMMDD e.g. 20260115
    full_date       DATE        NOT NULL UNIQUE,
    day_of_week     SMALLINT    NOT NULL,          -- 1=Monday, 7=Sunday (ISO)
    day_name        VARCHAR(10) NOT NULL,
    day_of_month    SMALLINT    NOT NULL,
    day_of_year     SMALLINT    NOT NULL,
    week_of_year    SMALLINT    NOT NULL,
    month_number    SMALLINT    NOT NULL,
    month_name      VARCHAR(10) NOT NULL,
    month_short     CHAR(3)     NOT NULL,
    quarter_number  SMALLINT    NOT NULL,
    quarter_name    CHAR(2)     NOT NULL,          -- 'Q1' .. 'Q4'
    year_number     INT         NOT NULL,
    year_month      CHAR(7)     NOT NULL,          -- '2026-01'
    fiscal_year     INT         NOT NULL,          -- FreshMart FY starts Feb 1
    fiscal_quarter  SMALLINT    NOT NULL,
    is_weekend      BOOLEAN     NOT NULL,
    is_holiday      BOOLEAN     NOT NULL DEFAULT FALSE,
    holiday_name    VARCHAR(50)
);

-- ── CUSTOMER DIMENSION (SCD Type 2) ──────────────────────────
-- WHY Type 2: Marketing analysis requires knowing which customer
-- segment and region the customer belonged to AT THE TIME OF
-- PURCHASE, not just their current segment.

CREATE TABLE dim_customer (
    customer_key        SERIAL      PRIMARY KEY,   -- Surrogate key (warehouse-generated)
    customer_id         INT         NOT NULL,      -- Natural key (source OLTP system)
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    phone               VARCHAR(20),
    segment             VARCHAR(50),               -- 'Premium', 'Standard', 'Budget'
    city                VARCHAR(100),
    state               VARCHAR(50),
    country             CHAR(2),
    postal_code         VARCHAR(10),
    registration_date   DATE,
    -- SCD Type 2 columns
    effective_start     DATE        NOT NULL,
    effective_end       DATE        NOT NULL DEFAULT '9999-12-31',
    is_current          BOOLEAN     NOT NULL DEFAULT TRUE,
    source_updated_at   TIMESTAMP                  -- Drives change detection in ETL
);

-- Index on the natural key + is_current for fast current-row lookups
CREATE INDEX idx_dim_customer_current  ON dim_customer(customer_id, is_current);
-- Index for SCD date-range lookups during fact table load
CREATE INDEX idx_dim_customer_daterange ON dim_customer(customer_id, effective_start, effective_end);

-- ── PRODUCT DIMENSION (SCD Type 2) ───────────────────────────
-- WHY Type 2: Finance needs historical unit_cost at the time of
-- sale for accurate margin calculations. Category changes must
-- be tracked so historical category-level analysis is accurate.

CREATE TABLE dim_product (
    product_key         SERIAL      PRIMARY KEY,
    product_id          INT         NOT NULL,
    product_name        VARCHAR(200) NOT NULL,
    brand_name          VARCHAR(100),
    subcategory_name    VARCHAR(100),
    category_name       VARCHAR(100),              -- Denormalized from 3 OLTP tables
    supplier_name       VARCHAR(200),
    unit_cost           DECIMAL(10,2),
    list_price          DECIMAL(10,2),
    weight_kg           DECIMAL(8,3),
    is_active           BOOLEAN     NOT NULL DEFAULT TRUE,
    -- SCD Type 2 columns
    effective_start     DATE        NOT NULL,
    effective_end       DATE        NOT NULL DEFAULT '9999-12-31',
    is_current          BOOLEAN     NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_dim_product_current ON dim_product(product_id, is_current);

-- ── STORE DIMENSION (SCD Type 1 — overwrite) ─────────────────
-- WHY Type 1: Store hierarchy changes are rare (a store rarely
-- moves regions). When they do occur, we want the new structure
-- reflected everywhere — we don't need to analyze "what region
-- was store X in before the reorganization?"

CREATE TABLE dim_store (
    store_key       SERIAL      PRIMARY KEY,
    store_id        INT         NOT NULL UNIQUE,
    store_name      VARCHAR(100) NOT NULL,
    store_type      VARCHAR(50),                   -- 'Supermarket', 'Express', 'Online'
    district_name   VARCHAR(100),
    region_name     VARCHAR(100),
    city            VARCHAR(100),
    state           VARCHAR(50),
    country         CHAR(2),
    postal_code     VARCHAR(10),
    square_footage  INT,
    opening_date    DATE,
    manager_name    VARCHAR(200)
);

-- ── PROMOTION DIMENSION ───────────────────────────────────────
CREATE TABLE dim_promotion (
    promotion_key   SERIAL      PRIMARY KEY,
    promotion_id    INT         NOT NULL UNIQUE,
    promotion_name  VARCHAR(200),
    promotion_type  VARCHAR(50),                   -- 'BOGO', 'Percentage', 'FixedAmount'
    discount_percent DECIMAL(5,2),
    start_date      DATE,
    end_date        DATE,
    is_active       BOOLEAN
);

-- Insert a "No Promotion" placeholder row (promotion_key = -1)
-- WHY: Fact rows without a promotion FK should point here, not NULL.
-- NULL on a FK makes reports harder to write (must handle NULL separately).
INSERT INTO dim_promotion (promotion_key, promotion_id, promotion_name, promotion_type, is_active)
VALUES (-1, -1, 'No Promotion', 'None', FALSE);
```

### Step 2.3: Create the Fact Table

```sql
-- ── FACT TABLE: SALES ─────────────────────────────────────────
-- GRAIN: One row per order line item (one product in one order)
-- Surrogate keys link to dimension tables.
-- Measures are pre-calculated to avoid runtime multiplication.

CREATE TABLE fact_sales (
    sales_key           BIGSERIAL   PRIMARY KEY,
    -- Dimension foreign keys
    date_key            INT         NOT NULL REFERENCES dim_date(date_key),
    customer_key        INT         NOT NULL REFERENCES dim_customer(customer_key),
    product_key         INT         NOT NULL REFERENCES dim_product(product_key),
    store_key           INT         NOT NULL REFERENCES dim_store(store_key),
    promotion_key       INT         NOT NULL DEFAULT -1 REFERENCES dim_promotion(promotion_key),
    -- Degenerate dimensions (no separate dim table warranted)
    order_id            INT         NOT NULL,
    order_line_number   SMALLINT    NOT NULL,
    -- Measures
    quantity            INT         NOT NULL,
    unit_price          DECIMAL(10,2) NOT NULL,
    unit_cost           DECIMAL(10,2) NOT NULL,
    discount_amount     DECIMAL(10,2) NOT NULL DEFAULT 0,
    gross_revenue       DECIMAL(12,2) NOT NULL,    -- quantity × unit_price
    net_revenue         DECIMAL(12,2) NOT NULL,    -- gross_revenue − discount_amount
    gross_margin        DECIMAL(12,2) NOT NULL,    -- net_revenue − (quantity × unit_cost)
    -- ETL audit
    etl_loaded_at       TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Performance indexes for the most common query patterns
-- Rule: index columns that appear in WHERE or JOIN ON clauses
CREATE INDEX idx_fact_date            ON fact_sales(date_key);
CREATE INDEX idx_fact_store_date      ON fact_sales(store_key, date_key);
CREATE INDEX idx_fact_product_date    ON fact_sales(product_key, date_key);
CREATE INDEX idx_fact_customer        ON fact_sales(customer_key);
-- Covering index for the most common dashboard query (avoids heap lookup)
CREATE INDEX idx_fact_date_store_cov
    ON fact_sales(date_key, store_key)
    INCLUDE (net_revenue, quantity, gross_margin);
```

---

## PART 3: ETL — POPULATE DIMENSIONS (90 min)

### Step 3.1: Populate the Date Dimension

**WHY:** The date dimension is loaded once and maintained manually. It covers historical data and a planning horizon. GENERATE_SERIES makes this a single SQL statement — no Python loop required.

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 3.1: Populate dim_date (2020-01-01 to 2030-12-31)
-- ═══════════════════════════════════════════════════════════════

INSERT INTO dim_date
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT                 AS date_key,
    d                                            AS full_date,
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
    END                                          AS fiscal_year,
    CASE
        WHEN EXTRACT(MONTH FROM d) IN (2,3,4)   THEN 1
        WHEN EXTRACT(MONTH FROM d) IN (5,6,7)   THEN 2
        WHEN EXTRACT(MONTH FROM d) IN (8,9,10)  THEN 3
        ELSE 4
    END::SMALLINT                                AS fiscal_quarter,
    EXTRACT(ISODOW FROM d) IN (6,7)            AS is_weekend,
    FALSE                                        AS is_holiday,
    NULL::VARCHAR(50)                            AS holiday_name
FROM GENERATE_SERIES(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) AS d;

-- Mark US public holidays (fixed-date only for simplicity)
UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'New Year''s Day'
WHERE month_number = 1  AND day_of_month = 1;

UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Independence Day'
WHERE month_number = 7  AND day_of_month = 4;

UPDATE dim_date SET is_holiday = TRUE, holiday_name = 'Christmas Day'
WHERE month_number = 12 AND day_of_month = 25;

-- Verify
SELECT COUNT(*) AS days_generated, MIN(full_date), MAX(full_date) FROM dim_date;
-- Expected: 4018 days, 2020-01-01, 2030-12-31
```

### Step 3.2: Load dim_store and dim_promotion (Type 1 — Simple UPSERT)

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 3.2: Load Type 1 dimensions — simple UPSERT
-- Type 1 = overwrite. No versioning needed.
-- ═══════════════════════════════════════════════════════════════

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
    district_name  = EXCLUDED.district_name,
    region_name    = EXCLUDED.region_name,
    manager_name   = EXCLUDED.manager_name;
    -- Type 1: all updates silently overwrite
```

### Step 3.3: SCD Type 2 MERGE for dim_customer

**WHY:** This is the heart of dimensional ETL. Correctly implementing SCD Type 2 ensures that when a customer changes segment from "Standard" to "Premium" in March, their purchases before March are still attributed to Standard, and purchases from March onward to Premium. This is the guarantee that makes historical analysis trustworthy.

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 3.3: SCD Type 2 ETL for dim_customer
-- Run daily during the nightly ETL window.
-- ═══════════════════════════════════════════════════════════════

-- Stage the current state from the OLTP source
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

-- STEP A: Close rows that have changed
-- Set effective_end to yesterday, mark as no longer current.
UPDATE dim_customer dc
SET
    effective_end = CURRENT_DATE - 1,
    is_current    = FALSE
FROM stg_customer stg
WHERE dc.customer_id = stg.customer_id
  AND dc.is_current  = TRUE
  AND (
      -- Use IS DISTINCT FROM to handle NULLs correctly in comparisons
      dc.segment      IS DISTINCT FROM stg.segment
      OR dc.city      IS DISTINCT FROM stg.city
      OR dc.state     IS DISTINCT FROM stg.state
      OR dc.email     IS DISTINCT FROM stg.email
  );

-- STEP B: Insert new versions for changed records AND brand-new customers
-- A record needs a new version if no is_current=TRUE row exists
-- (either it was just closed above, or it's a new customer entirely).
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
    CURRENT_DATE        AS effective_start,
    '9999-12-31'::DATE  AS effective_end,
    TRUE                AS is_current,
    stg.source_updated_at
FROM stg_customer stg
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer dc
    WHERE dc.customer_id = stg.customer_id
      AND dc.is_current  = TRUE
);

-- Verify: every source customer_id should have exactly ONE is_current=TRUE row
SELECT customer_id, COUNT(*) AS current_row_count
FROM dim_customer
WHERE is_current = TRUE
GROUP BY customer_id
HAVING COUNT(*) > 1;
-- Expected: 0 rows (no duplicates among current versions)
```

---

## PART 4: ETL — POPULATE THE FACT TABLE (45 min)

### Step 4.1: Incremental Fact Load with SCD-Aware Key Lookup

**WHY:** Full reloads of 15M rows nightly would take hours. The high-water mark pattern loads only records modified since the last run. The SCD-aware JOIN is the subtle but critical piece — it finds the dimension version that was active on the order date, not the current version.

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 4.1: Incremental fact_sales load
-- High-water mark = MAX(etl_loaded_at) in the fact table.
-- SCD-aware joins use date-range overlap, not is_current = TRUE.
-- ═══════════════════════════════════════════════════════════════

DO $$
DECLARE
    v_hwm TIMESTAMP;
BEGIN
    -- Get the last successful load timestamp
    SELECT COALESCE(MAX(etl_loaded_at), '1970-01-01'::TIMESTAMP)
    INTO   v_hwm
    FROM   fact_sales;

    RAISE NOTICE 'Loading fact_sales from high-water mark: %', v_hwm;

    INSERT INTO fact_sales (
        date_key, customer_key, product_key, store_key, promotion_key,
        order_id, order_line_number,
        quantity, unit_price, unit_cost, discount_amount,
        gross_revenue, net_revenue, gross_margin
    )
    SELECT
        -- Date dimension key (YYYYMMDD integer)
        TO_CHAR(o.order_date, 'YYYYMMDD')::INT          AS date_key,

        -- SCD-aware customer lookup: find the customer VERSION
        -- that was active on the order date — NOT the current version.
        -- This is the critical join: o.order_date must fall within
        -- the customer version's effective date range.
        dc.customer_key,

        -- SCD-aware product lookup (same logic)
        dp.product_key,

        ds.store_key,

        COALESCE(dpr.promotion_key, -1)                 AS promotion_key,

        o.order_id,
        oi.line_number                                   AS order_line_number,

        -- Measures (pre-calculated here to avoid runtime arithmetic)
        oi.quantity,
        oi.unit_price,
        p.unit_cost,
        COALESCE(oi.discount_amount, 0)                 AS discount_amount,
        (oi.quantity * oi.unit_price)                   AS gross_revenue,
        (oi.quantity * oi.unit_price
            - COALESCE(oi.discount_amount, 0))          AS net_revenue,
        (oi.quantity * oi.unit_price
            - COALESCE(oi.discount_amount, 0)
            - oi.quantity * p.unit_cost)                AS gross_margin

    FROM source_oltp.orders       o
    JOIN source_oltp.order_items  oi  ON o.order_id    = oi.order_id
    JOIN source_oltp.products     p   ON oi.product_id = p.product_id

    -- SCD-aware dimension lookups using date range overlap
    JOIN dim_customer dc
        ON  o.customer_id     = dc.customer_id
        AND o.order_date      BETWEEN dc.effective_start AND dc.effective_end

    JOIN dim_product dp
        ON  oi.product_id     = dp.product_id
        AND o.order_date      BETWEEN dp.effective_start AND dp.effective_end

    JOIN dim_store ds
        ON  o.store_id        = ds.store_id

    LEFT JOIN source_oltp.promotion_products pp
        ON  oi.product_id     = pp.product_id
    LEFT JOIN dim_promotion dpr
        ON  pp.promotion_id   = dpr.promotion_id
        AND o.order_date      BETWEEN dpr.start_date AND dpr.end_date

    -- Incremental filter: only new/modified orders since last run
    WHERE o.updated_at > v_hwm;

    RAISE NOTICE 'Inserted % rows into fact_sales', (
        SELECT COUNT(*) FROM fact_sales WHERE etl_loaded_at > v_hwm
    );
END $$;
```

---

## PART 5: ANALYTICAL QUERIES — PROVE THE VALUE (60 min)

### Step 5.1: The Same Report, Now < 2 Seconds

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.1: The 45-minute query, now < 2 seconds
-- 8 JOINs → 3 JOINs. Text GROUP BY → integer key GROUP BY.
-- ═══════════════════════════════════════════════════════════════

SELECT
    ds.region_name,
    ds.store_name,
    dp.category_name,
    dp.product_name,
    dd.year_month                              AS sales_month,
    COUNT(DISTINCT fs.order_id)                AS order_count,
    SUM(fs.quantity)                           AS units_sold,
    SUM(fs.gross_revenue)                      AS gross_revenue,
    SUM(fs.discount_amount)                    AS total_discounts,
    SUM(fs.net_revenue)                        AS net_revenue
FROM      fact_sales   fs
JOIN dim_date     dd ON fs.date_key    = dd.date_key
JOIN dim_store    ds ON fs.store_key   = ds.store_key
JOIN dim_product  dp ON fs.product_key = dp.product_key
WHERE dd.year_number = 2025
GROUP BY ds.region_name, ds.store_name, dp.category_name,
         dp.product_name, dd.year_month
ORDER BY ds.region_name, ds.store_name, dd.year_month;
```

### Step 5.2: Year-Over-Year Store Growth

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.2: YoY revenue growth using LAG window function
-- ═══════════════════════════════════════════════════════════════

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
ORDER BY cy.region_name, cy.store_name, cy.month_number;
```

### Step 5.3: Customer Cohort Retention Analysis

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.3: Cohort retention — which acquisition months retain best?
-- ═══════════════════════════════════════════════════════════════

WITH customer_cohorts AS (
    SELECT
        dc.customer_key,
        DATE_TRUNC('month', dc.registration_date)::DATE AS cohort_month
    FROM dim_customer dc
    WHERE dc.is_current = TRUE
),
active_by_month AS (
    SELECT DISTINCT
        fs.customer_key,
        DATE_TRUNC('month', dd.full_date)::DATE AS activity_month
    FROM      fact_sales fs
    JOIN dim_date dd ON fs.date_key = dd.date_key
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
        COUNT(DISTINCT cc.customer_key)     AS active_customers
    FROM customer_cohorts cc
    JOIN active_by_month  ab ON cc.customer_key = ab.customer_key
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

### Step 5.4: Running Totals & Moving Averages

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.4: Daily revenue with YTD running total and moving averages
-- ═══════════════════════════════════════════════════════════════

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
ORDER BY dd.full_date;
```

### Step 5.5: Pareto ABC Product Classification

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.5: Which 20% of products generate 80% of revenue?
-- Pareto/ABC analysis using cumulative window function.
-- ═══════════════════════════════════════════════════════════════

WITH product_revenue AS (
    SELECT
        dp.product_name,
        dp.category_name,
        SUM(fs.net_revenue) AS total_revenue
    FROM      fact_sales   fs
    JOIN dim_product dp ON fs.product_key = dp.product_key
    JOIN dim_date    dd ON fs.date_key    = dd.date_key
    WHERE dd.year_number = 2025
      AND dp.is_current  = TRUE
    GROUP BY dp.product_name, dp.category_name
),
ranked AS (
    SELECT
        product_name,
        category_name,
        total_revenue,
        SUM(total_revenue) OVER (ORDER BY total_revenue DESC)   AS cumulative_revenue,
        SUM(total_revenue) OVER ()                              AS grand_total,
        ROW_NUMBER()       OVER (ORDER BY total_revenue DESC)   AS rank,
        COUNT(*)           OVER ()                              AS total_products
    FROM product_revenue
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
LIMIT 100;
```

---

## PART 6: PERFORMANCE OPTIMIZATION (45 min)

### Step 6.1: Diagnose, Index, and Verify

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 6.1: Systematic performance optimization
-- ═══════════════════════════════════════════════════════════════

-- 1. Baseline measurement — run this BEFORE adding any indexes
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
JOIN dim_date  dd ON fs.date_key  = dd.date_key
WHERE dd.year_number = 2025
GROUP BY ds.region_name;

-- WHAT YOU WILL SEE BEFORE OPTIMIZATION:
-- Seq Scan on fact_sales — full table scan (slowest possible)
-- Hash Join materializing large intermediate results
-- Actual rows >> estimated rows (stale statistics)

-- 2. Update statistics so the planner makes better decisions
ANALYZE fact_sales;
ANALYZE dim_date;
ANALYZE dim_store;

-- 3. Add a covering index for the date filter + store join
-- INCLUDE adds extra columns to the index leaf pages.
-- This enables an Index Only Scan: no heap access needed at all.
CREATE INDEX idx_fact_date_store_covering
    ON fact_sales(date_key, store_key)
    INCLUDE (net_revenue, quantity, gross_margin);

-- 4. Re-run EXPLAIN — you should now see:
-- Index Only Scan using idx_fact_date_store_covering
-- Dramatically reduced actual time
EXPLAIN (ANALYZE, BUFFERS)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
JOIN dim_date  dd ON fs.date_key  = dd.date_key
WHERE dd.year_number = 2025
GROUP BY ds.region_name;
```

### Step 6.2: Materialized View for the BI Dashboard

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 6.2: Materialized view for the most expensive BI query
-- Refreshed nightly after the ETL completes.
-- ═══════════════════════════════════════════════════════════════

CREATE MATERIALIZED VIEW mv_daily_store_category_sales AS
SELECT
    fs.date_key,
    dd.year_month,
    dd.year_number,
    dd.month_number,
    dd.fiscal_year,
    dd.fiscal_quarter,
    ds.store_key,
    ds.store_name,
    ds.region_name,
    ds.district_name,
    dp.category_name,
    COUNT(*)                    AS line_item_count,
    COUNT(DISTINCT fs.order_id) AS order_count,
    SUM(fs.quantity)            AS total_units,
    SUM(fs.gross_revenue)       AS gross_revenue,
    SUM(fs.net_revenue)         AS net_revenue,
    SUM(fs.gross_margin)        AS gross_margin,
    SUM(fs.discount_amount)     AS total_discounts
FROM      fact_sales   fs
JOIN dim_date    dd ON fs.date_key    = dd.date_key
JOIN dim_store   ds ON fs.store_key   = ds.store_key
JOIN dim_product dp ON fs.product_key = dp.product_key
GROUP BY
    fs.date_key, dd.year_month, dd.year_number, dd.month_number,
    dd.fiscal_year, dd.fiscal_quarter,
    ds.store_key, ds.store_name, ds.region_name, ds.district_name,
    dp.category_name;

-- Create a unique index to enable CONCURRENT refresh (no read locks)
CREATE UNIQUE INDEX idx_mv_daily_sales
    ON mv_daily_store_category_sales(date_key, store_key, category_name);

-- Refresh nightly after ETL (add to your cron / Airflow DAG)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_store_category_sales;

-- Verify it replaces the base table query with a fast scan
EXPLAIN SELECT region_name, SUM(net_revenue)
FROM mv_daily_store_category_sales
WHERE year_number = 2025
GROUP BY region_name;
-- Expected: Seq Scan on mv (small) vs Index Scan on fact_sales (15M rows)
```

### Step 6.3: Partitioning Strategy

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 6.3: Range partitioning by year (for future growth)
-- Partition pruning means queries with a year filter skip all
-- other partitions entirely.
-- ═══════════════════════════════════════════════════════════════

-- Create the partitioned parent table
CREATE TABLE fact_sales_partitioned (
    LIKE fact_sales INCLUDING ALL          -- Copy all columns + constraints
) PARTITION BY RANGE (date_key);

-- Create yearly partitions
CREATE TABLE fact_sales_2023 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20230101) TO (20240101);

CREATE TABLE fact_sales_2024 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20240101) TO (20250101);

CREATE TABLE fact_sales_2025 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20250101) TO (20260101);

-- Default partition catches anything outside explicit ranges
CREATE TABLE fact_sales_default PARTITION OF fact_sales_partitioned
    DEFAULT;

-- Verify partition pruning is working
EXPLAIN SELECT COUNT(*) FROM fact_sales_partitioned
WHERE date_key BETWEEN 20250101 AND 20251231;
-- Expected: Append with only fact_sales_2025 scanned (others pruned)
```

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The fact table grain is "one row per order line item." What would break if you chose "one row per order" instead? Give a specific example of a business question that would produce wrong numbers with the coarser grain.

2. The `dim_promotion` table has a placeholder row with `promotion_key = -1`. Why is this better than allowing `NULL` on the `promotion_key` FK in `fact_sales`?

3. In the SCD Type 2 ETL, the fact table load uses `o.order_date BETWEEN dc.effective_start AND dc.effective_end` to find the right customer version. What happens if a late-arriving order arrives today for a purchase made 6 months ago, and the customer changed segments 3 months ago? Walk through the logic.

4. The materialized view `mv_daily_store_category_sales` pre-aggregates at the category level. A BI developer asks you to add product-level drill-through to the dashboard. What are the trade-offs of adding `product_name` to the materialized view GROUP BY?

5. Year-range partitioning was chosen for `fact_sales`. What alternative partitioning strategy would you consider if FreshMart added 50 new stores (making per-store analysis the most common query pattern), and why?
