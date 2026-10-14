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

> **About the data in this lab:** you build a deterministic *sample* of FreshMart's OLTP system — 60 stores, 1,200 products, 20,000 customers, 100,000 orders and 300,235 order lines (2023–2025). Every number below is exactly what you will see. The schema, the problems and the fixes are the same as at full production scale; only the timings are smaller.

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
├── 0. SOURCE SYSTEM (schema source_oltp)
│   └── 15 normalized OLTP tables + deterministic seed data
│
├── 1. SOURCE ANALYSIS
│   └── OLTP schema inspection + slow query diagnosis
│
├── 2. DIMENSIONAL MODEL (Star Schema, schema public)
│   ├── dim_date          (2020–2030, fiscal calendar, holidays)
│   ├── dim_customer      (SCD Type 2 — tracks segment, address & email changes)
│   ├── dim_product       (SCD Type 2 — tracks category, cost & price changes)
│   ├── dim_store         (SCD Type 1 — overwrite)
│   ├── dim_promotion     (SCD Type 1 — overwrite, with a "No Promotion" row)
│   └── fact_sales        (grain: one row per order line item)
│
├── 3. ETL SQL
│   ├── dim_date population (GENERATE_SERIES)
│   ├── Type 1 UPSERTs for dim_store and dim_promotion
│   ├── SCD Type 2 close-then-insert for dim_customer and dim_product
│   ├── Incremental, idempotent fact_sales load (high-water mark + ON CONFLICT)
│   └── Surrogate key lookups with SCD-aware date-range joins
│
├── 4. ANALYTICAL QUERIES
│   ├── Monthly sales by region/category (the 45-min query, now fast)
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

- **PostgreSQL 16** running in Docker exactly as set up in [Granular Lab 00](../Granular%20Labs/Lab_00_Environment_Setup.md): container `sk02-postgres`, database `freshmart`, user/password `postgres`/`postgres`, port 5432, and `C:\DataEngineering\sk02\sql` mounted at `/sql`. (Any PostgreSQL 14+ works; adjust the commands if you use a native install.)
- A SQL client: psql (inside the container), DBeaver, pgAdmin, or VS Code SQL extension
- New to SQL or data modeling? Work through the [Granular Labs](../Granular%20Labs/) 00–08 first — they teach every technique used here, step by step, on this exact schema.
- Optional: Access to a cloud data warehouse (Redshift, Snowflake, BigQuery) to run the platform-variant notes

**How to run each script in this lab:** save the SQL block as the file named in its first comment line under `C:\DataEngineering\sk02\sql\`, then run it from PowerShell:

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/<file>.sql
```

Ad-hoc queries can be run in an interactive session: `docker exec -it sk02-postgres psql -U postgres -d freshmart`.

---

## PART 0: BUILD THE SOURCE SYSTEM (30 min)

### Step 0.1: Create the OLTP Schema

**WHY:** A warehouse is always built *from* something. This creates FreshMart's operational database in its own schema, `source_oltp`, so the warehouse tables (schema `public`) stay clearly separated from the source. Note the snowflaked hierarchies (store → district → region, product → subcategory → category) that make reporting slow.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 01_source_oltp_ddl.sql — FreshMart OLTP source system (3NF)
-- Rerunnable: drops and recreates the whole source_oltp schema.
-- ═══════════════════════════════════════════════════════════════

DROP SCHEMA IF EXISTS source_oltp CASCADE;
CREATE SCHEMA source_oltp;

-- ── Store hierarchy: region → district → store ────────────────
CREATE TABLE source_oltp.store_regions (
    region_id       SERIAL       PRIMARY KEY,
    region_name     VARCHAR(50)  NOT NULL UNIQUE
);

CREATE TABLE source_oltp.store_districts (
    district_id     SERIAL       PRIMARY KEY,
    district_name   VARCHAR(100) NOT NULL,
    region_id       INT          NOT NULL REFERENCES source_oltp.store_regions(region_id)
);

CREATE TABLE source_oltp.employees (
    employee_id     SERIAL       PRIMARY KEY,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    job_title       VARCHAR(50)  NOT NULL,
    hire_date       DATE         NOT NULL
);

CREATE TABLE source_oltp.stores (
    store_id        SERIAL       PRIMARY KEY,
    store_name      VARCHAR(100) NOT NULL,
    store_type      VARCHAR(50)  NOT NULL,           -- 'Supermarket' | 'Express' | 'Online'
    district_id     INT          NOT NULL REFERENCES source_oltp.store_districts(district_id),
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(50)  NOT NULL,
    country         CHAR(2)      NOT NULL DEFAULT 'US',
    postal_code     VARCHAR(10)  NOT NULL,
    square_footage  INT,                             -- NULL for the online store
    opening_date    DATE         NOT NULL,
    manager_id      INT          REFERENCES source_oltp.employees(employee_id),
    updated_at      TIMESTAMP    NOT NULL DEFAULT now()
);

-- ── Product hierarchy: category → subcategory → product ───────
CREATE TABLE source_oltp.product_categories (
    category_id     SERIAL       PRIMARY KEY,
    category_name   VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE source_oltp.product_subcats (
    subcategory_id   SERIAL       PRIMARY KEY,
    subcategory_name VARCHAR(100) NOT NULL,
    category_id      INT          NOT NULL REFERENCES source_oltp.product_categories(category_id)
);

CREATE TABLE source_oltp.suppliers (
    supplier_id     SERIAL       PRIMARY KEY,
    supplier_name   VARCHAR(200) NOT NULL
);

CREATE TABLE source_oltp.products (
    product_id      SERIAL        PRIMARY KEY,
    product_name    VARCHAR(200)  NOT NULL,
    brand_name      VARCHAR(100),
    subcategory_id  INT           NOT NULL REFERENCES source_oltp.product_subcats(subcategory_id),
    supplier_id     INT           REFERENCES source_oltp.suppliers(supplier_id),
    unit_cost       DECIMAL(10,2) NOT NULL CHECK (unit_cost  >= 0),
    list_price      DECIMAL(10,2) NOT NULL CHECK (list_price >= 0),
    weight_kg       DECIMAL(8,3),
    is_active       BOOLEAN       NOT NULL DEFAULT TRUE,
    updated_at      TIMESTAMP     NOT NULL DEFAULT now()
);

-- ── Customers ─────────────────────────────────────────────────
CREATE TABLE source_oltp.customer_segments (
    segment_id      SERIAL       PRIMARY KEY,
    segment_name    VARCHAR(50)  NOT NULL UNIQUE     -- 'Premium' | 'Standard' | 'Budget'
);

CREATE TABLE source_oltp.customers (
    customer_id       SERIAL       PRIMARY KEY,
    first_name        VARCHAR(100) NOT NULL,
    last_name         VARCHAR(100) NOT NULL,
    email             VARCHAR(255),                  -- nullable: not every shopper gives one
    phone             VARCHAR(20),
    segment_id        INT          REFERENCES source_oltp.customer_segments(segment_id),
    registration_date DATE         NOT NULL,
    updated_at        TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE TABLE source_oltp.customer_addresses (
    address_id      SERIAL       PRIMARY KEY,
    customer_id     INT          NOT NULL REFERENCES source_oltp.customers(customer_id),
    city            VARCHAR(100) NOT NULL,
    state           VARCHAR(50)  NOT NULL,
    country         CHAR(2)      NOT NULL DEFAULT 'US',
    postal_code     VARCHAR(10)  NOT NULL,
    is_primary      BOOLEAN      NOT NULL DEFAULT TRUE
);
-- At most one primary address per customer
CREATE UNIQUE INDEX uq_customer_primary_address
    ON source_oltp.customer_addresses(customer_id) WHERE is_primary;

-- ── Promotions ────────────────────────────────────────────────
CREATE TABLE source_oltp.promotions (
    promotion_id     SERIAL       PRIMARY KEY,
    promotion_name   VARCHAR(200) NOT NULL,
    promotion_type   VARCHAR(50)  NOT NULL CHECK (promotion_type IN ('BOGO','Percentage','FixedAmount')),
    discount_percent DECIMAL(5,2),                   -- NULL for FixedAmount promotions
    start_date       DATE         NOT NULL,
    end_date         DATE         NOT NULL,
    is_active        BOOLEAN      NOT NULL,
    CHECK (end_date >= start_date)
);

-- Which products a promotion applies to (many-to-many)
CREATE TABLE source_oltp.promotion_products (
    promotion_id    INT NOT NULL REFERENCES source_oltp.promotions(promotion_id),
    product_id      INT NOT NULL REFERENCES source_oltp.products(product_id),
    PRIMARY KEY (promotion_id, product_id)
);

-- ── Sales transactions ────────────────────────────────────────
CREATE TABLE source_oltp.orders (
    order_id        SERIAL       PRIMARY KEY,
    customer_id     INT          NOT NULL REFERENCES source_oltp.customers(customer_id),
    store_id        INT          NOT NULL REFERENCES source_oltp.stores(store_id),
    order_date      DATE         NOT NULL,
    status          VARCHAR(20)  NOT NULL DEFAULT 'completed'
                    CHECK (status IN ('completed','cancelled','returned')),
    updated_at      TIMESTAMP    NOT NULL DEFAULT now()   -- drives incremental ETL
);

CREATE TABLE source_oltp.order_items (
    order_id        INT           NOT NULL REFERENCES source_oltp.orders(order_id),
    line_number     SMALLINT      NOT NULL,
    product_id      INT           NOT NULL REFERENCES source_oltp.products(product_id),
    quantity        INT           NOT NULL CHECK (quantity > 0),
    unit_price      DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),  -- price actually charged
    discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    promotion_id    INT           REFERENCES source_oltp.promotions(promotion_id),  -- promotion actually applied (NULL = none)
    PRIMARY KEY (order_id, line_number)
);

-- ── Helper: deterministic "random" numbers ────────────────────
-- random() gives different data every run. This function always returns the
-- same number in [0, 1) for the same (seed, salt), so everyone's data — and
-- every expected output in the labs — is identical.
CREATE FUNCTION source_oltp.pseudo_random(seed BIGINT, salt TEXT)
RETURNS NUMERIC
LANGUAGE sql IMMUTABLE
AS $$
    SELECT ('x' || substr(md5(salt || ':' || seed::text), 1, 8))::bit(32)::bigint / 4294967296.0
$$;
```

**Expected output:** a `NOTICE: schema "source_oltp" does not exist, skipping` on the very first run, then `DROP SCHEMA`, `CREATE SCHEMA`, 15 × `CREATE TABLE`, one `CREATE INDEX` and `CREATE FUNCTION`. The script is rerunnable — it rebuilds the source from scratch (rerun Step 0.2 afterwards).

### Step 0.2: Seed Deterministic Data

**WHY:** Realistic volumes expose real problems (fan-out, slow scans, SCD edge cases). `pseudo_random()` instead of `random()` makes every machine produce identical data, so your results match this guide exactly. Expect it to take 1–2 minutes.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 02_source_oltp_seed.sql — deterministic FreshMart source data
-- Run after 01_source_oltp_ddl.sql. Same data on every machine.
-- Volumes: 60 stores, 1,200 products, 20,000 customers,
--          100,000 orders (2023-01-01 .. 2025-12-31), ~300,000 order lines.
-- ═══════════════════════════════════════════════════════════════

-- Shorthand used below:  R(x, 'salt') = source_oltp.pseudo_random(x, 'salt')

-- ── Regions and districts (4 regions × 3 districts) ───────────
INSERT INTO source_oltp.store_regions (region_name)
VALUES ('Northeast'), ('Southeast'), ('Midwest'), ('West');

INSERT INTO source_oltp.store_districts (district_name, region_id)
SELECT c.city || ' District', 1 + (d - 1) / 3
FROM generate_series(1, 12) AS d
JOIN (VALUES (1,'New York'),(2,'Boston'),(3,'Philadelphia'),
             (4,'Atlanta'),(5,'Miami'),(6,'Charlotte'),
             (7,'Chicago'),(8,'Detroit'),(9,'Minneapolis'),
             (10,'Los Angeles'),(11,'Seattle'),(12,'Denver')) AS c(id, city)
  ON c.id = d
ORDER BY d;

-- ── Employees (first 60 are store managers) ───────────────────
INSERT INTO source_oltp.employees (first_name, last_name, job_title, hire_date)
SELECT
    (ARRAY['James','Mary','John','Patricia','Robert','Jennifer','Michael','Linda','David','Elizabeth'])[1 + e % 10],
    (ARRAY['Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis','Rodriguez','Martinez','Lopez','Wilson'])[1 + (e * 7) % 12],
    CASE WHEN e <= 60 THEN 'Store Manager' ELSE 'Sales Associate' END,
    DATE '2012-01-01' + floor(source_oltp.pseudo_random(e, 'hire') * 3650)::int
FROM generate_series(1, 300) AS e;

-- ── Stores (60, spread over the 12 districts) ─────────────────
INSERT INTO source_oltp.stores (store_name, store_type, district_id, city, state, country,
                                postal_code, square_footage, opening_date, manager_id, updated_at)
SELECT
    c.city || ' #' || s,
    st.store_type,
    c.id,
    c.city, c.state, 'US', c.postal_code,
    CASE st.store_type
        WHEN 'Supermarket' THEN 40000 + floor(source_oltp.pseudo_random(s, 'sqft') * 20000)::int
        WHEN 'Express'     THEN  8000 + floor(source_oltp.pseudo_random(s, 'sqft') *  4000)::int
    END,
    DATE '2010-01-01' + floor(source_oltp.pseudo_random(s, 'open') * 4000)::int,
    s,                                              -- employee s manages store s
    TIMESTAMP '2023-01-01 00:00:00'
FROM generate_series(1, 60) AS s
CROSS JOIN LATERAL (SELECT CASE WHEN s % 10 = 0 THEN 'Online'
                                WHEN s % 3  = 0 THEN 'Express'
                                ELSE 'Supermarket' END AS store_type) AS st
JOIN (VALUES (1,'New York','NY','10001'),(2,'Boston','MA','02108'),(3,'Philadelphia','PA','19103'),
             (4,'Atlanta','GA','30303'),(5,'Miami','FL','33130'),(6,'Charlotte','NC','28202'),
             (7,'Chicago','IL','60601'),(8,'Detroit','MI','48226'),(9,'Minneapolis','MN','55401'),
             (10,'Los Angeles','CA','90012'),(11,'Seattle','WA','98101'),(12,'Denver','CO','80202'))
     AS c(id, city, state, postal_code)
  ON c.id = 1 + (s - 1) % 12
ORDER BY s;

-- ── Product hierarchy (6 categories × 4 subcategories) ────────
INSERT INTO source_oltp.product_categories (category_name)
VALUES ('Produce'), ('Dairy'), ('Bakery'), ('Beverages'), ('Snacks'), ('Household');

INSERT INTO source_oltp.product_subcats (subcategory_name, category_id)
VALUES ('Fruit',1),('Vegetables',1),('Herbs',1),('Salads',1),
       ('Milk',2),('Cheese',2),('Yogurt',2),('Butter',2),
       ('Bread',3),('Pastries',3),('Cakes',3),('Tortillas',3),
       ('Water',4),('Soda',4),('Juice',4),('Coffee',4),
       ('Chips',5),('Cookies',5),('Nuts',5),('Candy',5),
       ('Cleaning',6),('Paper Goods',6),('Laundry',6),('Kitchenware',6);

INSERT INTO source_oltp.suppliers (supplier_name)
SELECT 'Supplier ' || lpad(sp::text, 2, '0') FROM generate_series(1, 20) AS sp;

-- ── Products (1,200; every 25th is discontinued) ──────────────
INSERT INTO source_oltp.products (product_name, brand_name, subcategory_id, supplier_id,
                                  unit_cost, list_price, weight_kg, is_active, updated_at)
SELECT
    b.brand || ' ' || ps.subcategory_name || ' ' || p,
    b.brand,
    ps.subcategory_id,
    1 + (p * 7) % 20,
    round(pr.list_price * (0.55 + source_oltp.pseudo_random(p, 'cost') * 0.25), 2),
    pr.list_price,
    round(0.1 + source_oltp.pseudo_random(p, 'weight') * 4.9, 3),
    p % 25 <> 0,
    TIMESTAMP '2023-01-01 00:00:00'
FROM generate_series(1, 1200) AS p
JOIN source_oltp.product_subcats ps ON ps.subcategory_id = 1 + (p - 1) % 24
CROSS JOIN LATERAL (SELECT (ARRAY['FreshFarm','GoodLife','Sunny','Prairie','BlueRiver',
                                  'Harvest','Coastal','Summit','Maple','Urban'])[1 + p % 10] AS brand) AS b
CROSS JOIN LATERAL (SELECT round(1.50 + source_oltp.pseudo_random(p, 'price') * 48.50, 2) AS list_price) AS pr
ORDER BY p;

-- ── Customers (20,000) ────────────────────────────────────────
INSERT INTO source_oltp.customer_segments (segment_name)
VALUES ('Premium'), ('Standard'), ('Budget');

INSERT INTO source_oltp.customers (first_name, last_name, email, phone, segment_id,
                                   registration_date, updated_at)
SELECT
    n.first_name,
    n.last_name,
    CASE WHEN source_oltp.pseudo_random(c, 'email') < 0.15 THEN NULL     -- ~15% gave no email
         ELSE lower(n.first_name || '.' || n.last_name || c || '@example.com') END,
    '555-' || lpad((c % 10000)::text, 4, '0'),
    CASE WHEN source_oltp.pseudo_random(c, 'segment') < 0.15 THEN 1      -- Premium  ~15%
         WHEN source_oltp.pseudo_random(c, 'segment') < 0.70 THEN 2      -- Standard ~55%
         ELSE 3 END,                                                     -- Budget   ~30%
    r.registration_date,
    r.registration_date::timestamp
FROM generate_series(1, 20000) AS c
CROSS JOIN LATERAL (SELECT
    (ARRAY['Olivia','Liam','Emma','Noah','Ava','Elijah','Sophia','Lucas','Isabella','Mason',
           'Mia','Ethan','Amelia','Logan','Harper','James','Evelyn','Aiden','Abigail','Jackson'])[1 + c % 20] AS first_name,
    (ARRAY['Smith','Johnson','Williams','Brown','Jones','Garcia','Miller','Davis','Rodriguez','Martinez',
           'Hernandez','Lopez','Gonzalez','Wilson','Anderson','Thomas','Taylor','Moore','Jackson','Martin',
           'Lee','Perez','Thompson'])[1 + (c * 7) % 23] AS last_name) AS n
CROSS JOIN LATERAL (SELECT DATE '2021-01-01'
                           + floor(source_oltp.pseudo_random(c, 'registered') * 1461)::int AS registration_date) AS r
ORDER BY c;

-- One primary address per customer, plus a non-primary one for every 10th customer
INSERT INTO source_oltp.customer_addresses (customer_id, city, state, country, postal_code, is_primary)
SELECT a.customer_id, c.city, c.state, 'US', c.postal_code, a.is_primary
FROM (
    SELECT cu AS customer_id, TRUE AS is_primary,
           1 + floor(source_oltp.pseudo_random(cu, 'home') * 12)::int AS city_id
    FROM generate_series(1, 20000) AS cu
    UNION ALL
    SELECT cu, FALSE, 1 + floor(source_oltp.pseudo_random(cu, 'work') * 12)::int
    FROM generate_series(10, 20000, 10) AS cu
) AS a
JOIN (VALUES (1,'New York','NY','10001'),(2,'Boston','MA','02108'),(3,'Philadelphia','PA','19103'),
             (4,'Atlanta','GA','30303'),(5,'Miami','FL','33130'),(6,'Charlotte','NC','28202'),
             (7,'Chicago','IL','60601'),(8,'Detroit','MI','48226'),(9,'Minneapolis','MN','55401'),
             (10,'Los Angeles','CA','90012'),(11,'Seattle','WA','98101'),(12,'Denver','CO','80202'))
     AS c(id, city, state, postal_code)
  ON c.id = a.city_id
ORDER BY a.customer_id, a.is_primary DESC;

-- ── Promotions (40 two-week promotions, one every 27 days) ────
INSERT INTO source_oltp.promotions (promotion_name, promotion_type, discount_percent,
                                    start_date, end_date, is_active)
SELECT
    'Promo ' || lpad(pm::text, 2, '0') || ' - ' || t.promotion_type,
    t.promotion_type,
    CASE t.promotion_type WHEN 'BOGO'       THEN 50.00
                          WHEN 'Percentage' THEN 10.00 + 5 * (pm % 4)
                          ELSE NULL END,
    DATE '2023-01-02' + (pm - 1) * 27,
    DATE '2023-01-02' + (pm - 1) * 27 + 13,
    DATE '2023-01-02' + (pm - 1) * 27 + 13 >= DATE '2025-12-01'
FROM generate_series(1, 40) AS pm
CROSS JOIN LATERAL (SELECT (ARRAY['Percentage','BOGO','FixedAmount'])[1 + pm % 3] AS promotion_type) AS t
ORDER BY pm;

-- Each promotion covers 60 products
INSERT INTO source_oltp.promotion_products (promotion_id, product_id)
SELECT pm, 1 + (pm * 37 + k * 20) % 1200
FROM generate_series(1, 40) AS pm
CROSS JOIN generate_series(0, 59) AS k;

CREATE INDEX idx_promotion_products_product ON source_oltp.promotion_products(product_id);

-- ── Orders (100,000; never before the customer registered) ────
INSERT INTO source_oltp.orders (customer_id, store_id, order_date, status, updated_at)
SELECT
    x.customer_id,
    1 + floor(source_oltp.pseudo_random(o, 'store') * 60)::int,
    x.order_date,
    CASE WHEN source_oltp.pseudo_random(o, 'status') < 0.04 THEN 'cancelled'
         WHEN source_oltp.pseudo_random(o, 'status') < 0.06 THEN 'returned'
         ELSE 'completed' END,
    x.order_date + floor(source_oltp.pseudo_random(o, 'time') * 86400)::int * INTERVAL '1 second'
FROM generate_series(1, 100000) AS o
CROSS JOIN LATERAL (SELECT 1 + floor(source_oltp.pseudo_random(o, 'customer') * 20000)::int AS customer_id) AS cu
JOIN source_oltp.customers c ON c.customer_id = cu.customer_id
CROSS JOIN LATERAL (
    SELECT cu.customer_id,
           GREATEST(c.registration_date, DATE '2023-01-01')
           + floor(source_oltp.pseudo_random(o, 'date')
                   * (DATE '2025-12-31' - GREATEST(c.registration_date, DATE '2023-01-01') + 1))::int AS order_date
) AS x
ORDER BY o;

-- ── Order items (1–5 lines per order) ─────────────────────────
INSERT INTO source_oltp.order_items (order_id, line_number, product_id, quantity,
                                     unit_price, discount_amount, promotion_id)
SELECT
    o.order_id,
    l.line_number,
    p.product_id,
    q.quantity,
    p.list_price,
    CASE pr.promotion_type
        WHEN 'Percentage'  THEN round(q.quantity * p.list_price * pr.discount_percent / 100, 2)
        WHEN 'BOGO'        THEN round((q.quantity / 2) * p.list_price, 2)   -- every 2nd item free
        WHEN 'FixedAmount' THEN LEAST(1.00 * q.quantity, q.quantity * p.list_price)
        ELSE 0
    END,
    pr.promotion_id
FROM source_oltp.orders o
CROSS JOIN LATERAL generate_series(1, 1 + floor(source_oltp.pseudo_random(o.order_id, 'lines') * 5)::int) AS l(line_number)
CROSS JOIN LATERAL (SELECT 1 + floor(source_oltp.pseudo_random(o.order_id * 10 + l.line_number, 'product') * 1200)::int AS product_id,
                           1 + floor(source_oltp.pseudo_random(o.order_id * 10 + l.line_number, 'qty') * 4)::int AS quantity) AS q
JOIN source_oltp.products p ON p.product_id = q.product_id
LEFT JOIN LATERAL (
    SELECT pm.promotion_id, pm.promotion_type, pm.discount_percent
    FROM source_oltp.promotion_products pp
    JOIN source_oltp.promotions pm ON pm.promotion_id = pp.promotion_id
    WHERE pp.product_id = q.product_id
      AND o.order_date BETWEEN pm.start_date AND pm.end_date
    ORDER BY pm.promotion_id
    LIMIT 1
) AS pr ON TRUE
ORDER BY o.order_id, l.line_number;

-- Refresh planner statistics (and pg_stat_user_tables row estimates)
ANALYZE;
```

**Expected output:**

```
INSERT 0 4
INSERT 0 12
INSERT 0 300
INSERT 0 60
INSERT 0 6
INSERT 0 24
INSERT 0 20
INSERT 0 1200
INSERT 0 3
INSERT 0 20000
INSERT 0 22000
INSERT 0 40
INSERT 0 2400
CREATE INDEX
INSERT 0 100000
INSERT 0 300235
ANALYZE
```

---

## PART 1: UNDERSTAND THE PROBLEM — SOURCE ANALYSIS (45 min)

### Step 1.1: Inspect the OLTP Schema

**WHY:** Before designing a warehouse, you must understand the source. Knowing the OLTP schema tells you which normalizations to undo, which surrogate keys exist, and which tables hold the data you need.

**HOW:**

```sql
-- ═══════════════════════════════════════════════════════════════
-- PART 1: OLTP SOURCE ANALYSIS
-- FreshMart operational PostgreSQL database (schema source_oltp)
-- ═══════════════════════════════════════════════════════════════

-- Step 1.1: Understand data volumes
SELECT
    relname          AS table_name,
    n_live_tup       AS estimated_row_count,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE schemaname = 'source_oltp'
ORDER BY n_live_tup DESC;

-- Expected row counts in the FreshMart sample OLTP
-- (total_size varies slightly between machines):
-- order_items:            300,235
-- orders:                 100,000
-- customer_addresses:      22,000
-- customers:               20,000
-- promotion_products:       2,400
-- products:                 1,200
-- employees:                  300
-- stores:                      60
-- promotions:                  40
-- product_subcats:             24
-- suppliers:                   20
-- store_districts:             12
-- product_categories:           6
-- store_regions:                4
-- customer_segments:            3
```

> If `estimated_row_count` shows 0 for some tables, the statistics collector hasn't flushed yet — wait a few seconds and rerun, or run `ANALYZE;`.

### Step 1.2: Run the Current "Simple" Sales Report and Diagnose It

**WHY:** EXPLAIN ANALYZE is your diagnostic tool. Before writing a single line of DW code, you need to understand exactly why the current query is slow. This gives you the before picture for your performance proof.

```sql
-- Step 1.2: Diagnose the slow query
-- This is the query that takes 45 minutes on the production OLTP database
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
FROM source_oltp.orders             o
JOIN source_oltp.order_items        oi ON o.order_id       = oi.order_id
JOIN source_oltp.products           p  ON oi.product_id    = p.product_id
JOIN source_oltp.product_subcats    ps ON p.subcategory_id = ps.subcategory_id
JOIN source_oltp.product_categories pc ON ps.category_id   = pc.category_id
JOIN source_oltp.stores             s  ON o.store_id       = s.store_id
JOIN source_oltp.store_districts    sd ON s.district_id    = sd.district_id
JOIN source_oltp.store_regions      sr ON sd.region_id     = sr.region_id
WHERE o.order_date >= DATE '2025-01-01'
  AND o.order_date <  DATE '2026-01-01'        -- last full year of data
GROUP BY
    s.store_name, sr.region_name, pc.category_name,
    p.product_name, DATE_TRUNC('month', o.order_date)
ORDER BY sr.region_name, s.store_name, sales_month;

-- WHAT YOU WILL SEE IN THE PLAN:
-- Seq Scan on order_items and orders — no index on order_date or the FK columns
-- A chain of Hash Joins materializing large intermediate result sets
-- A Sort + GroupAggregate over text columns (no surrogate keys), producing 122,481 groups
-- On production volumes, all of this competes with live transactions for I/O

-- WHY THIS FAILS:
-- 1. 8 JOINs across normalized tables (the real system has 30+)
-- 2. Missing indexes on FK columns
-- 3. Full table scans on every access
-- 4. GROUP BY forces a sort on many rows of string data
-- 5. OLTP database is not the right tool for this workload
```

**Expected output:** a plan whose last lines show `Planning Time: … ms` and `Execution Time: … ms`. The exact timings vary by machine; the *shape* (Seq Scans, Hash Joins, Sort, GroupAggregate) is what matters. Remove `EXPLAIN (...)` to see the actual report: 122,481 rows.

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
-- BUSINESS RULE: only 'completed' orders are loaded (cancelled/returned are not revenue)

-- SCHEMA TYPE: Star (not snowflake)
-- WHY: Fewer joins for BI tools, better for Power BI/Tableau DAX
-- Trade-off: Some dimension redundancy (category_name stored in dim_product)

-- SCD STRATEGY:
--   dim_customer: Type 2 (track segment, city/state & email changes for historical accuracy)
--   dim_product:  Type 2 (track category, cost & price changes for margin analysis)
--   dim_store:    Type 1 (overwrite — hierarchy rarely changes; no analytical need for history)
--   dim_promotion: Type 1 (overwrite — promotions don't retroactively change)
--   Effective dates are INCLUSIVE: a closed version ends the day BEFORE the new one starts,
--   so "order_date BETWEEN effective_start AND effective_end" matches exactly one version.

-- CONFORMED DIMENSIONS:
--   dim_date is the same table used by all future fact tables in this warehouse
--   dim_store is the same table used by fact_sales and any future fact_inventory
--   This enables cross-fact-table analysis without ETL duplication
```

### Step 2.2 & 2.3: Create the Dimension Tables and the Fact Table

**WHY:** Surrogate keys (`SERIAL`) decouple the warehouse from source IDs and make SCD Type 2 possible (one customer, many versions). The `-1` "No Promotion" row means fact rows never hold NULL foreign keys. The `UNIQUE (order_id, order_line_number)` constraint on the fact table enforces the grain — a reload can never create duplicates. The covering index for dashboards is deliberately **not** created yet: Part 6 measures the before/after.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 03_dw_ddl.sql — FreshMart star schema (warehouse tables live in
-- the default "public" schema; the source lives in "source_oltp").
-- Rerunnable: drops warehouse objects first (children before parents).
-- ═══════════════════════════════════════════════════════════════

DROP MATERIALIZED VIEW IF EXISTS mv_daily_store_category_sales;
DROP TABLE IF EXISTS fact_sales_partitioned CASCADE;
DROP TABLE IF EXISTS fact_sales, dim_date, dim_customer, dim_product,
                     dim_store, dim_promotion, etl_control CASCADE;

-- ── DATE DIMENSION ────────────────────────────────────────────
CREATE TABLE dim_date (
    date_key        INT         PRIMARY KEY,       -- YYYYMMDD e.g. 20250115
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
    year_month      CHAR(7)     NOT NULL,          -- '2025-01'
    fiscal_year     INT         NOT NULL,          -- FreshMart FY starts Feb 1
    fiscal_quarter  SMALLINT    NOT NULL,
    is_weekend      BOOLEAN     NOT NULL,
    is_holiday      BOOLEAN     NOT NULL DEFAULT FALSE,
    holiday_name    VARCHAR(50)
);

-- ── CUSTOMER DIMENSION (SCD Type 2) ──────────────────────────
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
    source_updated_at   TIMESTAMP
);
CREATE INDEX idx_dim_customer_current   ON dim_customer(customer_id, is_current);
CREATE INDEX idx_dim_customer_daterange ON dim_customer(customer_id, effective_start, effective_end);
-- Guard: at most ONE current version per customer
CREATE UNIQUE INDEX uq_dim_customer_one_current ON dim_customer(customer_id) WHERE is_current;

-- ── PRODUCT DIMENSION (SCD Type 2) ───────────────────────────
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
CREATE INDEX idx_dim_product_current   ON dim_product(product_id, is_current);
CREATE INDEX idx_dim_product_daterange ON dim_product(product_id, effective_start, effective_end);
CREATE UNIQUE INDEX uq_dim_product_one_current ON dim_product(product_id) WHERE is_current;

-- ── STORE DIMENSION (SCD Type 1 — overwrite) ─────────────────
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

-- ── PROMOTION DIMENSION (SCD Type 1) ─────────────────────────
CREATE TABLE dim_promotion (
    promotion_key    SERIAL      PRIMARY KEY,
    promotion_id     INT         NOT NULL UNIQUE,
    promotion_name   VARCHAR(200),
    promotion_type   VARCHAR(50),                  -- 'BOGO', 'Percentage', 'FixedAmount'
    discount_percent DECIMAL(5,2),
    start_date       DATE,
    end_date         DATE,
    is_active        BOOLEAN
);

-- "No Promotion" placeholder row (promotion_key = -1).
-- Fact rows without a promotion point here instead of holding NULL.
INSERT INTO dim_promotion (promotion_key, promotion_id, promotion_name, promotion_type, is_active)
VALUES (-1, -1, 'No Promotion', 'None', FALSE);

-- ── FACT TABLE: SALES ─────────────────────────────────────────
-- GRAIN: one row per order line item (one product in one order)
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
    etl_loaded_at       TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- One fact row per source order line: makes reloads idempotent
    CONSTRAINT uq_fact_sales_order_line UNIQUE (order_id, order_line_number)
);

-- Foreign-key / filter indexes (the covering index is added in Part 6)
CREATE INDEX idx_fact_date            ON fact_sales(date_key);
CREATE INDEX idx_fact_store_date      ON fact_sales(store_key, date_key);
CREATE INDEX idx_fact_product_date    ON fact_sales(product_key, date_key);
CREATE INDEX idx_fact_customer        ON fact_sales(customer_key);

-- ── ETL CONTROL TABLE (high-water marks) ─────────────────────
CREATE TABLE etl_control (
    table_name      TEXT        PRIMARY KEY,
    high_water_mark TIMESTAMP   NOT NULL
);
INSERT INTO etl_control VALUES ('fact_sales', '1970-01-01');
```

**Expected output:** `NOTICE`s about objects that don't exist yet (first run only), then `CREATE TABLE` / `CREATE INDEX` confirmations and two `INSERT 0 1` (the `-1` promotion and the `etl_control` seed row).

---

## PART 3: ETL — POPULATE DIMENSIONS (90 min)

### Step 3.1: Populate the Date Dimension

**WHY:** The date dimension is loaded once and maintained manually. It covers historical data and a planning horizon. GENERATE_SERIES makes this a single SQL statement — no Python loop required. `ON CONFLICT DO NOTHING` makes the script safe to rerun.

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

### Step 3.2: Load dim_store and dim_promotion (Type 1 — Simple UPSERT)

**WHY:** Type 1 = overwrite. `ON CONFLICT (natural key) DO UPDATE SET col = EXCLUDED.col` inserts new members and overwrites changed ones in a single idempotent statement. Note how the store hierarchy (district, region) and the manager's name are *denormalized* into one wide dimension row.

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

**Expected output:**

```
INSERT 0 60
INSERT 0 40
 stores | promotions_incl_placeholder
--------+-----------------------------
     60 |                          41
(1 row)
```

(Rerunning reports the same `INSERT 0 60` / `INSERT 0 40` — those rows are *upserted*, the totals stay 60 and 41.)

### Step 3.3: SCD Type 2 Load for dim_customer (close-then-insert)

**WHY:** This is the heart of dimensional ETL. Correctly implementing SCD Type 2 ensures that when a customer changes segment from "Standard" to "Premium" in March, their purchases before March are still attributed to Standard, and purchases from March onward to Premium. This is the guarantee that makes historical analysis trustworthy.

Two details matter:
- The **first** version of a customer is valid from `1900-01-01` ("the beginning of time"). If it started today, the date-range lookup in Part 4 would find no customer version for any historical order and the fact load would silently load nothing.
- `IS DISTINCT FROM` is the NULL-safe "not equal" — `NULL <> 'x'` is NULL (not true), so a plain `<>` would miss changes to or from NULL emails.

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

### Step 3.4: SCD Type 2 Load for dim_product

**WHY:** Finance needs the `unit_cost` that applied *at the time of sale* for accurate margins, and category reorganisations must not rewrite history. Same pattern as customers; untracked attributes (name, brand, supplier, weight, active flag) are overwritten Type 1-style in Step C.

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

**Expected output (first run):** the same shape as Step 3.3 with `SELECT 1200`, `UPDATE 0`, `INSERT 0 1200`, `UPDATE 0`, `(0 rows)` and `1200 | 1200`.

---

## PART 4: ETL — POPULATE THE FACT TABLE (45 min)

### Step 4.1: Incremental, Idempotent Fact Load with SCD-Aware Key Lookup

**WHY:** Full reloads of 15M rows nightly would take hours. The high-water mark pattern loads only orders modified since the last run. The SCD-aware JOIN is the subtle but critical piece — it finds the dimension version that was active on the order date, not the current version.

Production details built in:
- The high-water mark lives in `etl_control` and is compared with the **source's** `updated_at` (same clock on both sides). Its new value is frozen *before* the load and advanced in the same transaction as the insert.
- The promotion comes from `order_items.promotion_id` (the promotion actually applied), so a product that belongs to several promotions cannot multiply fact rows.
- `ON CONFLICT (order_id, order_line_number) DO UPDATE` makes reruns and late updates safe: a line is updated, never duplicated.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 08_load_fact_sales.sql — incremental, idempotent fact_sales load
-- High-water mark = MAX(source orders.updated_at) already loaded,
-- stored in etl_control. SCD-aware joins use the order date.
-- A DO block runs as ONE transaction: load + HWM advance are atomic.
-- ═══════════════════════════════════════════════════════════════

DO $$
DECLARE
    v_hwm      TIMESTAMP;   -- where the last run stopped
    v_new_hwm  TIMESTAMP;   -- where this run will stop
    v_rows     BIGINT;
BEGIN
    SELECT high_water_mark INTO v_hwm
    FROM   etl_control
    WHERE  table_name = 'fact_sales';

    -- Freeze the upper bound BEFORE loading, so rows arriving during the
    -- load are picked up next time instead of being skipped.
    SELECT MAX(updated_at) INTO v_new_hwm
    FROM   source_oltp.orders
    WHERE  updated_at > v_hwm;

    RAISE NOTICE 'Loading fact_sales: updated_at in (%, %]', v_hwm, v_new_hwm;

    IF v_new_hwm IS NULL THEN
        RAISE NOTICE 'Nothing new to load.';
        RETURN;
    END IF;

    INSERT INTO fact_sales (
        date_key, customer_key, product_key, store_key, promotion_key,
        order_id, order_line_number,
        quantity, unit_price, unit_cost, discount_amount,
        gross_revenue, net_revenue, gross_margin
    )
    SELECT
        TO_CHAR(o.order_date, 'YYYYMMDD')::INT          AS date_key,
        dc.customer_key,                                -- version valid on the order date
        dp.product_key,                                 -- version valid on the order date
        ds.store_key,
        COALESCE(dpr.promotion_key, -1)                 AS promotion_key,
        o.order_id,
        oi.line_number                                  AS order_line_number,
        oi.quantity,
        oi.unit_price,
        dp.unit_cost,                                   -- cost as it was at time of sale
        oi.discount_amount,
        oi.quantity * oi.unit_price                     AS gross_revenue,
        oi.quantity * oi.unit_price - oi.discount_amount AS net_revenue,
        oi.quantity * oi.unit_price - oi.discount_amount
            - oi.quantity * dp.unit_cost                AS gross_margin
    FROM source_oltp.orders       o
    JOIN source_oltp.order_items  oi ON o.order_id = oi.order_id

    -- SCD-aware dimension lookups using the order date
    JOIN dim_customer dc
        ON  o.customer_id = dc.customer_id
        AND o.order_date  BETWEEN dc.effective_start AND dc.effective_end
    JOIN dim_product dp
        ON  oi.product_id = dp.product_id
        AND o.order_date  BETWEEN dp.effective_start AND dp.effective_end
    JOIN dim_store ds
        ON  o.store_id    = ds.store_id
    -- The promotion actually applied to the line (at most one → no fan-out)
    LEFT JOIN dim_promotion dpr
        ON  oi.promotion_id = dpr.promotion_id

    WHERE o.updated_at >  v_hwm
      AND o.updated_at <= v_new_hwm
      AND o.status = 'completed'            -- business rule: revenue = completed orders

    -- Idempotency: a line already in the fact is updated, never duplicated
    ON CONFLICT (order_id, order_line_number) DO UPDATE SET
        date_key        = EXCLUDED.date_key,
        customer_key    = EXCLUDED.customer_key,
        product_key     = EXCLUDED.product_key,
        store_key       = EXCLUDED.store_key,
        promotion_key   = EXCLUDED.promotion_key,
        quantity        = EXCLUDED.quantity,
        unit_price      = EXCLUDED.unit_price,
        unit_cost       = EXCLUDED.unit_cost,
        discount_amount = EXCLUDED.discount_amount,
        gross_revenue   = EXCLUDED.gross_revenue,
        net_revenue     = EXCLUDED.net_revenue,
        gross_margin    = EXCLUDED.gross_margin,
        etl_loaded_at   = CURRENT_TIMESTAMP;

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    UPDATE etl_control
    SET    high_water_mark = v_new_hwm
    WHERE  table_name = 'fact_sales';

    RAISE NOTICE 'Upserted % rows into fact_sales; high-water mark now %', v_rows, v_new_hwm;
END $$;

-- Reconciliation: source (completed lines) vs warehouse — must match exactly
SELECT
    (SELECT COUNT(*)
       FROM source_oltp.order_items oi
       JOIN source_oltp.orders o ON o.order_id = oi.order_id
      WHERE o.status = 'completed')                                  AS source_lines,
    (SELECT COUNT(*) FROM fact_sales)                                AS fact_lines,
    (SELECT SUM(oi.quantity * oi.unit_price - oi.discount_amount)
       FROM source_oltp.order_items oi
       JOIN source_oltp.orders o ON o.order_id = oi.order_id
      WHERE o.status = 'completed')                                  AS source_net_revenue,
    (SELECT SUM(net_revenue) FROM fact_sales)                        AS fact_net_revenue;
```

**Expected output (first run):**

```
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Loading fact_sales: updated_at in (1970-01-01 00:00:00, 2025-12-31 23:56:01]
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Upserted 282259 rows into fact_sales; high-water mark now 2025-12-31 23:56:01
DO
 source_lines | fact_lines | source_net_revenue | fact_net_revenue
--------------+------------+--------------------+------------------
       282259 |     282259 |        17315871.37 |      17315871.37
(1 row)
```

**Run it again immediately.** Expected: `NOTICE: Loading fact_sales: updated_at in (2025-12-31 23:56:01, <NULL>]`, `NOTICE: Nothing new to load.`, and an unchanged reconciliation row. That is idempotency.

### Step 4.2: Simulate the Next Day and Rerun the Nightly ETL

**WHY:** SCD Type 2 and incremental loading only prove themselves when the source changes. This script plays "one day in the OLTP": 40 customers change segment, 17 customers move to Denver, 12 products get a price rise, product 7 moves from Yogurt (Dairy) to Salads (Produce), store 1 gets a new manager, and customer 500 places a new order today.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 09_simulate_day2_changes.sql — "a day passes in the OLTP system"
-- Run ONCE, after the first full ETL run. Then rerun scripts 05–08.
-- NOT idempotent on purpose (it is a test fixture, not ETL).
-- ═══════════════════════════════════════════════════════════════

BEGIN;

-- 1. 40 customers change segment (Premium ↔ Standard) → SCD2 on dim_customer
UPDATE source_oltp.customers
SET segment_id = CASE WHEN segment_id = 1 THEN 2 ELSE 1 END,
    updated_at = now()
WHERE customer_id % 500 = 0;

-- 2. 20 customers move their primary address to Denver → SCD2 on dim_customer
UPDATE source_oltp.customer_addresses
SET city = 'Denver', state = 'CO', postal_code = '80202'
WHERE is_primary AND customer_id % 1000 = 7 AND city <> 'Denver';

-- 3. 12 products get a 10% price increase → SCD2 on dim_product
UPDATE source_oltp.products
SET list_price = round(list_price * 1.10, 2),
    updated_at = now()
WHERE product_id % 100 = 1;          -- products 1, 101, 201, ... 1101

-- 4. Product 7 is recategorised (Yogurt → Salads, i.e. Dairy → Produce) → SCD2 on dim_product
UPDATE source_oltp.products
SET subcategory_id = 4, updated_at = now()
WHERE product_id = 7;

-- 5. Store 1 gets a new manager → Type 1 overwrite on dim_store
UPDATE source_oltp.stores
SET manager_id = 62, updated_at = now()
WHERE store_id = 1;

-- 6. A new order arrives today for customer 500 (who just changed segment)
INSERT INTO source_oltp.orders (customer_id, store_id, order_date, status, updated_at)
VALUES (500, 1, CURRENT_DATE, 'completed', now());

INSERT INTO source_oltp.order_items (order_id, line_number, product_id, quantity, unit_price, discount_amount)
SELECT currval('source_oltp.orders_order_id_seq'), l.line_number, l.product_id, 2, p.list_price, 0
FROM (VALUES (1, 101), (2, 7)) AS l(line_number, product_id)
JOIN source_oltp.products p ON p.product_id = l.product_id;

COMMIT;
```

**Expected output:** `BEGIN`, `UPDATE 40`, `UPDATE 17`, `UPDATE 12`, `UPDATE 1`, `UPDATE 1`, `INSERT 0 1`, `INSERT 0 2`, `COMMIT`.

Now rerun the nightly chain in order — `05_load_type1_dims.sql`, `06_scd2_dim_customer.sql`, `07_scd2_dim_product.sql`, `08_load_fact_sales.sql`. **Expected:**

| Script | Key lines |
|---|---|
| 05 | `INSERT 0 60`, `INSERT 0 40` (store 1's manager overwritten in place) |
| 06 | `UPDATE 57` (versions closed), `INSERT 0 57` (new versions), totals `20057 \| 20000` |
| 07 | `UPDATE 13`, `INSERT 0 13`, totals `1213 \| 1200` |
| 08 | `Upserted 2 rows`; reconciliation `282261 \| 282261 \| 17316002.29 \| 17316002.29` |

### Step 4.3: Verify History Is Preserved

```sql
-- Customer 500 now has two versions...
SELECT customer_key, segment, city, effective_start, effective_end, is_current
FROM dim_customer WHERE customer_id = 500 ORDER BY effective_start;

-- ...and each fact row points at the version valid on its order date
SELECT dc.segment, COUNT(*) AS lines, SUM(fs.net_revenue) AS net_revenue
FROM fact_sales fs
JOIN dim_customer dc ON fs.customer_key = dc.customer_key
WHERE dc.customer_id = 500
GROUP BY dc.segment ORDER BY dc.segment;

-- Store 1 (Type 1): the new manager replaced the old one — no history kept
SELECT store_id, store_name, manager_name FROM dim_store WHERE store_id = 1;
```

**Expected output** (`<yesterday>`/`<today>` are your run dates; new versions get keys 20001–20057 in `customer_id` order, so customer 500's is 20002):

```
 customer_key | segment |   city    | effective_start | effective_end | is_current
--------------+---------+-----------+-----------------+---------------+------------
          500 | Budget  | Charlotte | 1900-01-01      | <yesterday>   | f
        20002 | Premium | Charlotte | <today>         | 9999-12-31    | t

 segment | lines | net_revenue
---------+-------+-------------
 Budget  |    12 |      717.39
 Premium |     2 |      130.92

 store_id | store_name  | manager_name
----------+-------------+---------------
        1 | New York #1 | John Williams
```

All 12 historical lines stay attributed to *Budget*; only today's order counts as *Premium*.

---

## PART 5: ANALYTICAL QUERIES — PROVE THE VALUE (60 min)

Save the five queries of this part together as `C:\DataEngineering\sk02\sql\10_analytics.sql`, or run them one at a time in an interactive session.

### Step 5.1: The Same Report, Now on the Star Schema

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.1: The 45-minute query, now fast
-- 8 JOINs → 3 JOINs. Text hierarchy lookups → pre-joined dimensions.
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
ORDER BY ds.region_name, ds.store_name, dd.year_month, dp.category_name, dp.product_name;
```

**Expected output:** 115,705 rows (fewer than the 122,481 of Step 1.2 because only completed orders are revenue in the warehouse). Prefix it with `EXPLAIN ANALYZE` and compare the plan with Step 1.2: three joins to small dimensions instead of eight.

### Step 5.2: Year-Over-Year Store Growth

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.2: YoY revenue growth — self-join this year to last year
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

**Expected output:** 720 rows (60 stores × 12 months). The first three:

```
 store_name  | region_name | month_number | cy_revenue | py_revenue | yoy_growth_pct
-------------+-------------+--------------+------------+------------+----------------
 Chicago #19 | Midwest     |            1 |   10981.36 |    5675.91 |          93.47
 Chicago #19 | Midwest     |            2 |    7839.39 |    3646.41 |         114.99
 Chicago #19 | Midwest     |            3 |   12648.78 |    5586.84 |         126.40
```

(Growth is high because FreshMart's customer base kept growing through 2024.)

### Step 5.3: Customer Cohort Retention Analysis

**WHY:** A cohort is the group of customers whose **first purchase** fell in the same month. Period 0 is that month (100% by definition); `FIRST_VALUE` over the cohort's rows divides every later month by that starting size. Grouping by `customer_id` (the natural key) rather than `customer_key` counts purchases made under *any* SCD2 version of a customer.

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

**Expected output:** one row per cohort month and period (0–12). The first four rows:

```
 cohort_month | period_number | active_customers | retention_pct
--------------+---------------+------------------+---------------
 2023-01-01   |             0 |             1275 |         100.0
 2023-01-01   |             1 |              120 |           9.4
 2023-01-01   |             2 |              164 |          12.9
 2023-01-01   |             3 |              153 |          12.0
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

**Expected output:** 365 rows. The first three:

```
 full_date  | day_name  | daily_revenue | cumulative_ytd | rolling_7day_avg | rolling_30day_avg
------------+-----------+---------------+----------------+------------------+-------------------
 2025-01-01 | Wednesday |      14973.41 |       14973.41 |         14973.41 |          14973.41
 2025-01-02 | Thursday  |      22447.71 |       37421.12 |         18710.56 |          18710.56
 2025-01-03 | Friday    |      17372.47 |       54793.59 |         18264.53 |          18264.53
```

(The moving averages only differ from each other once more than 7 days are in the window.)

### Step 5.5: Pareto ABC Product Classification

**WHY:** Revenue is summed per `product_id` across *all* SCD2 versions of a product, then labelled with the product's *current* name and category — otherwise a product whose price or category changed would be split into several rows.

```sql
-- ═══════════════════════════════════════════════════════════════
-- STEP 5.5: Which 20% of products generate 80% of revenue?
-- Pareto/ABC analysis using cumulative window function.
-- ═══════════════════════════════════════════════════════════════

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
LIMIT 100;
```

**Expected output:** the top three rows are

```
 rank |     product_name      | category_name | revenue  | cumulative_pct |  abc_class  | product_percentile
------+-----------------------+---------------+----------+----------------+-------------+--------------------
    1 | Summit Nuts 907       | Snacks        | 14619.98 |            0.2 | A — Top 80% |                0.1
    2 | Sunny Kitchenware 912 | Household     | 14525.66 |            0.4 | A — Top 80% |                0.2
    3 | Summit Water 1117     | Beverages     | 14412.67 |            0.6 | A — Top 80% |                0.3
```

Without the `LIMIT`, all 1,200 products are classified: 684 × A, 277 × B, 239 × C. (Sample data is far flatter than real retail, where ~20% of products make 80% of revenue.)

---

## PART 6: PERFORMANCE OPTIMIZATION (45 min)

Save this part as `11_performance.sql`.

### Step 6.1: Diagnose, Index, and Verify

**WHY:** Measure first, change one thing, measure again. The query filters on the fact table's own `date_key` (a quarter of data — the typical dashboard range), so a covering index on `(date_key, store_key) INCLUDE (net_revenue, …)` can answer it without touching the table at all.

```sql
-- ═══════════════════════════════════════════════════════════════
-- 11_performance.sql — guided lab Part 6
-- ═══════════════════════════════════════════════════════════════

-- 6.1 Baseline (only the FK indexes from 03_dw_ddl.sql exist)
DROP INDEX IF EXISTS idx_fact_date_store_covering;
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
WHERE fs.date_key BETWEEN 20251001 AND 20251231      -- Q4 2025
GROUP BY ds.region_name;

ANALYZE fact_sales;
ANALYZE dim_date;
ANALYZE dim_store;

CREATE INDEX idx_fact_date_store_covering
    ON fact_sales(date_key, store_key)
    INCLUDE (net_revenue, quantity, gross_margin);

-- Index-only scans need an up-to-date visibility map → VACUUM once after bulk loads
VACUUM (ANALYZE) fact_sales;

EXPLAIN (ANALYZE, BUFFERS)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
WHERE fs.date_key BETWEEN 20251001 AND 20251231      -- Q4 2025
GROUP BY ds.region_name;
```

**WHAT YOU WILL SEE:**
- **Before:** `Seq Scan on fact_sales` (or a Bitmap Heap Scan using `idx_fact_date`) reading the table heap, feeding a Hash Join with `dim_store` and a HashAggregate.
- **After:** `Index Only Scan using idx_fact_date_store_covering on fact_sales` with `Heap Fetches: 0` — the answer comes entirely from the index. (`VACUUM` updates the visibility map; without it you would see many heap fetches.)
- The result is the same both times:

```
 region_name |  revenue
-------------+-----------
 Midwest     | 470619.53
 Northeast   | 471159.56
 Southeast   | 482082.61
 West        | 480016.26
```

### Step 6.2: Materialized View for the BI Dashboard

```sql
-- 6.2 Materialized view
DROP MATERIALIZED VIEW IF EXISTS mv_daily_store_category_sales;
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

CREATE UNIQUE INDEX idx_mv_daily_sales
    ON mv_daily_store_category_sales(date_key, store_key, category_name);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_store_category_sales;

SELECT COUNT(*) AS mv_rows FROM mv_daily_store_category_sales;

EXPLAIN SELECT region_name, SUM(net_revenue)
FROM mv_daily_store_category_sales
WHERE year_number = 2025
GROUP BY region_name;
```

**Expected output:** `SELECT 169128` from the `CREATE MATERIALIZED VIEW`, `CREATE INDEX`, `REFRESH MATERIALIZED VIEW`, `mv_rows = 169128`, and an `EXPLAIN` showing an aggregate over a (possibly parallel) `Seq Scan on mv_daily_store_category_sales` — a small pre-aggregated table instead of joins over 282,261 fact rows. `REFRESH ... CONCURRENTLY` (possible because of the unique index) lets dashboards keep reading while it refreshes; schedule it after every nightly ETL run.

### Step 6.3: Partitioning Strategy

**WHY:** On a partitioned table, the primary key must include the partition key, so `LIKE fact_sales INCLUDING ALL` (which would copy `PRIMARY KEY (sales_key)`) fails. We copy columns, defaults and checks, then add `PRIMARY KEY (sales_key, date_key)`.

```sql
-- 6.3 Partitioning
DROP TABLE IF EXISTS fact_sales_partitioned CASCADE;
CREATE TABLE fact_sales_partitioned (
    LIKE fact_sales INCLUDING DEFAULTS INCLUDING CONSTRAINTS
) PARTITION BY RANGE (date_key);

ALTER TABLE fact_sales_partitioned
    ADD CONSTRAINT pk_fact_sales_partitioned PRIMARY KEY (sales_key, date_key);

CREATE TABLE fact_sales_2023 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20230101) TO (20240101);
CREATE TABLE fact_sales_2024 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20240101) TO (20250101);
CREATE TABLE fact_sales_2025 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20250101) TO (20260101);
CREATE TABLE fact_sales_default PARTITION OF fact_sales_partitioned
    DEFAULT;

INSERT INTO fact_sales_partitioned SELECT * FROM fact_sales;
ANALYZE fact_sales_partitioned;

SELECT tableoid::regclass AS partition, COUNT(*)
FROM fact_sales_partitioned GROUP BY 1 ORDER BY 1;

EXPLAIN SELECT COUNT(*) FROM fact_sales_partitioned
WHERE date_key BETWEEN 20250101 AND 20251231;
```

**Expected output:**

```
INSERT 0 282261
ANALYZE
     partition      | count
--------------------+--------
 fact_sales_2023    |  59649
 fact_sales_2024    |  98430
 fact_sales_2025    | 124180
 fact_sales_default |      2
(4 rows)
```

and an `EXPLAIN` that scans `fact_sales_2025` only (`Aggregate` over `Seq Scan on fact_sales_2025`, or the parallel form `Finalize Aggregate` / `Gather` / `Parallel Seq Scan on fact_sales_2025`) — the 2023, 2024 and default partitions are pruned. (With a single surviving partition PostgreSQL drops the `Append` node entirely.) The 2 rows in the default partition are the day-2 order from Part 4 — in production you would create next year's partition in advance.

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The fact table grain is "one row per order line item." What would break if you chose "one row per order" instead? Give a specific example of a business question that would produce wrong numbers with the coarser grain.

2. The `dim_promotion` table has a placeholder row with `promotion_key = -1`. Why is this better than allowing `NULL` on the `promotion_key` FK in `fact_sales`?

3. In the SCD Type 2 ETL, the fact table load uses `o.order_date BETWEEN dc.effective_start AND dc.effective_end` to find the right customer version. What happens if a late-arriving order arrives today for a purchase made 6 months ago, and the customer changed segments 3 months ago? Walk through the logic. What would happen if the first version of every customer started at `CURRENT_DATE` instead of `1900-01-01`?

4. The materialized view `mv_daily_store_category_sales` pre-aggregates at the category level. A BI developer asks you to add product-level drill-through to the dashboard. What are the trade-offs of adding `product_name` to the materialized view GROUP BY?

5. Year-range partitioning was chosen for `fact_sales`. What alternative partitioning strategy would you consider if FreshMart added 50 new stores (making per-store analysis the most common query pattern), and why?
