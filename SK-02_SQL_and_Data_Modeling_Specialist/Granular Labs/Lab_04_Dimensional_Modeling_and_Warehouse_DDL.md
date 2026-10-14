# Lab 04 — Dimensional Modeling and the Warehouse DDL

> **Goal:** Design FreshMart's star schema (grain, facts, dimensions, keys, SCD types) and create it in PostgreSQL exactly as the guided lab does.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 03
> **Guided lab coverage:** PART 2 (Steps 2.1–2.3) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Contrast OLTP and OLAP designs, and star vs snowflake schemas.
- Declare a fact table's **grain** and choose facts (measures) that are true at that grain.
- Explain **surrogate** vs **natural** keys and why SCD Type 2 needs surrogates.
- Choose **SCD Type 1 or Type 2** per dimension, and explain inclusive `effective_start` / `effective_end` dates.
- Explain **conformed**, **degenerate** and **placeholder (−1 "unknown/none")** dimension members.
- Create `dim_date`, `dim_customer`, `dim_product`, `dim_store`, `dim_promotion`, `fact_sales` and `etl_control` with the right constraints and indexes.

**Key terms**

| Term | Meaning |
|---|---|
| **Fact table** | Measurements of a business event (a sale line): numbers you SUM. Long and narrow. |
| **Dimension table** | Context for the facts — who, what, where, when, which promotion. Short and wide. |
| **Star schema** | One fact table joined directly to each dimension (one hop). A **snowflake** keeps dimensions normalized (store → district → region) — more joins, so we don't. |
| **Grain** | Exactly what one fact row represents. Declared first, before any column. |
| **SCD** (Slowly Changing Dimension) | How a dimension handles attribute changes: **Type 1** overwrite (no history), **Type 2** new row per version (full history). |

## Steps

### Step 1 — Write the design decisions before any DDL

Create `C:\DataEngineering\sk02\design_notes.md` and write the four Kimball decisions in your own words:

```text
1. Business process : retail sales (order lines)
2. Grain            : one row per order line item (one product in one order, one store, one date)
                      — completed orders only
3. Dimensions       : date, customer, product, store, promotion
                      (+ degenerate: order_id, order_line_number)
4. Facts            : quantity, unit_price, unit_cost, discount_amount,
                      gross_revenue, net_revenue, gross_margin
```

**Expected output:** `Get-Content C:\DataEngineering\sk02\design_notes.md` prints your four decisions.

Why this order? Facts only make sense relative to a grain. "Order total" is *not* true for one line, so it is not a fact here (summing it over lines would double-count). Picking the **finest** grain (the line) means you can always roll up to order, store, day or customer — but never drill below the grain you chose.

✅ **Checkpoint:** your notes state the grain in one sentence.

### Step 2 — Pre-calculate the measures

The fact table stores ready-to-sum numbers so BI tools never multiply at query time:

- `gross_revenue = quantity × unit_price`
- `net_revenue = gross_revenue − discount_amount`
- `gross_margin = net_revenue − quantity × unit_cost` (cost **at the time of sale**)

Check the formulas on one real order from the source:

```sql
SELECT oi.line_number, p.product_name, oi.quantity, oi.unit_price, oi.discount_amount, p.unit_cost,
       oi.quantity * oi.unit_price                                        AS gross_revenue,
       oi.quantity * oi.unit_price - oi.discount_amount                   AS net_revenue,
       oi.quantity * oi.unit_price - oi.discount_amount - oi.quantity * p.unit_cost AS gross_margin
FROM source_oltp.order_items oi
JOIN source_oltp.products p ON p.product_id = oi.product_id
WHERE oi.order_id = 13616
ORDER BY oi.line_number;
```

**Expected output:**

```
 line_number |   product_name   | quantity | unit_price | discount_amount | unit_cost | gross_revenue | net_revenue | gross_margin
-------------+------------------+----------+------------+-----------------+-----------+---------------+-------------+--------------
           1 | Harvest Milk 125 |        1 |       3.41 |            0.00 |      2.12 |          3.41 |        3.41 |         1.29
           2 | Prairie Nuts 643 |        4 |      42.76 |            0.00 |     23.60 |        171.04 |      171.04 |        76.64
           3 | Prairie Nuts 643 |        2 |      42.76 |            0.00 |     23.60 |         85.52 |       85.52 |        38.32
(3 rows)
```

(Same product on two lines is normal — the grain is the *line*, so both rows become separate facts.)

✅ **Checkpoint:** you can recompute `gross_margin` for line 2 by hand: 171.04 − 4 × 23.60 = 76.64.

### Step 3 — Keys and Slowly Changing Dimensions

**Natural key** = the source's id (`customer_id = 42`). **Surrogate key** = a meaningless warehouse number (`customer_key`), generated with `SERIAL`. Why bother?

1. **SCD Type 2 needs it.** When customer 42 changes segment, the dimension keeps *two rows* for customer 42 — so `customer_id` can't be the primary key.
2. Sources get replaced or merged; surrogate keys insulate the warehouse.
3. Small integer keys join fast.

FreshMart's SCD decisions (the guided lab's Step 2.1):

| Dimension | Type | Tracked attributes | Why |
|---|---|---|---|
| `dim_customer` | 2 | segment, city, state, email | Marketing reports by segment/region *at the time of purchase* |
| `dim_product` | 2 | subcategory, category, unit_cost, list_price | Margin needs historical cost; category reorganisations must not rewrite history |
| `dim_store` | 1 | — | Hierarchy rarely changes; nobody analyses "region before the reorg" |
| `dim_promotion` | 1 | — | Promotions don't change retroactively |

Every Type 2 dimension gets three columns:

| customer_key | customer_id | segment | effective_start | effective_end | is_current |
|---|---|---|---|---|---|
| 500 | 500 | Budget | 1900-01-01 | 2026-03-09 | false |
| 20002 | 500 | Premium | 2026-03-10 | 9999-12-31 | true |

- `effective_end` is **inclusive** — the old version ends the day *before* the new one starts, so `order_date BETWEEN effective_start AND effective_end` matches exactly one version.
- `'9999-12-31'` (not NULL) for the open version keeps range checks simple.
- The **first** version starts at `1900-01-01` so every historical order finds it (Lab 05 explains why that matters).
- A **partial unique index** `ON dim_customer(customer_id) WHERE is_current` makes "two current versions" impossible — the same trick as Lab 01's one-primary-address index.

Prove the inclusive-dates rule to yourself — a day belongs to exactly one version:

```sql
SELECT d AS order_date,
       d BETWEEN DATE '1900-01-01' AND DATE '2026-03-09' AS in_old_version,
       d BETWEEN DATE '2026-03-10' AND DATE '9999-12-31' AS in_new_version
FROM (VALUES (DATE '2026-03-09'), (DATE '2026-03-10')) AS t(d);
```

**Expected output:**

```
 order_date | in_old_version | in_new_version
------------+----------------+----------------
 2026-03-09 | t              | f
 2026-03-10 | f              | t
(2 rows)
```

✅ **Checkpoint:** you can explain why `customer_id` must *not* be `UNIQUE` in `dim_customer`, but `store_id` *can* be `UNIQUE` in `dim_store`.

### Step 4 — Special members, conformed and degenerate dimensions

- **Placeholder row:** many sales have no promotion. Instead of a NULL foreign key, those facts point to `promotion_key = -1` ("No Promotion"). Reports then never need special NULL handling, and an inner join never silently drops them. The load uses `COALESCE(promotion_key, -1)`.
- **Conformed dimension:** one `dim_date` (and one `dim_store`) shared by every current and future fact table (e.g. `fact_inventory`) — so cross-process reports line up.
- **Degenerate dimension:** `order_id` and `order_line_number` live directly in the fact row (no table of their own) — for drill-down and audit. Together they are the grain's natural key, so the fact gets `UNIQUE (order_id, order_line_number)`: a reload can never duplicate a line.
- **ETL control table:** `etl_control` remembers the **high-water mark** (the latest source `updated_at` already loaded) — Lab 06 uses it.

See the NULL-key problem with three made-up fact rows (a `VALUES` list acts as a tiny inline table):

```sql
WITH facts(promotion_key) AS (VALUES (5), (NULL), (-1)),
     promos(promotion_key) AS (VALUES (5), (-1))
SELECT (SELECT COUNT(*) FROM facts)                                                    AS fact_rows,
       (SELECT COUNT(*) FROM facts f JOIN promos p ON p.promotion_key = f.promotion_key) AS rows_after_inner_join;
```

**Expected output:**

```
 fact_rows | rows_after_inner_join
-----------+-----------------------
         3 |                     2
(1 row)
```

The NULL-keyed row vanished from the join (NULL = anything is never true); the `-1` row survived because it matches the placeholder.

✅ **Checkpoint:** you can say what happens to a no-promotion sale in a report joined `INNER JOIN dim_promotion` if the FK were NULL instead of −1 (it disappears).

### Step 5 — The date dimension

Every warehouse has a date dimension: one row per calendar day with pre-computed attributes, so no report ever does date math. Its key is the date as an integer `YYYYMMDD` (`20250115`) — readable, compact, sorts correctly (the one accepted "meaningful" surrogate key). FreshMart's **fiscal year starts on 1 February**, so 15 Jan 2025 belongs to fiscal year 2024, fiscal quarter 4 — a classic reason a date table beats ad-hoc `EXTRACT`. It also carries `is_weekend`, `is_holiday` and `holiday_name`.

Try the key and fiscal-year logic on two days around the fiscal boundary:

```sql
SELECT TO_CHAR(d, 'YYYYMMDD')::INT AS date_key,
       EXTRACT(ISODOW FROM d)      AS iso_day_of_week,
       CASE WHEN EXTRACT(MONTH FROM d) >= 2 THEN EXTRACT(YEAR FROM d)::INT
            ELSE EXTRACT(YEAR FROM d)::INT - 1 END AS fiscal_year
FROM generate_series(DATE '2025-01-31', DATE '2025-02-01', INTERVAL '1 day') AS d;
```

**Expected output:**

```
 date_key | iso_day_of_week | fiscal_year
----------+-----------------+-------------
 20250131 |               5 |        2024
 20250201 |               6 |        2025
(2 rows)
```

✅ **Checkpoint:** you can state which fiscal year and quarter 1 Feb 2025 belongs to (FY 2025, Q1).

### Step 6 — Create the warehouse

Save the guided lab's Steps 2.2–2.3 DDL as `C:\DataEngineering\sk02\sql\03_dw_ddl.sql`. The warehouse lives in the default schema `public`; the source stays in `source_oltp`. Identify in it everything from Steps 3–5: `SERIAL` / `BIGSERIAL` surrogate keys, SCD2 columns and partial unique indexes, the `-1` row, `uq_fact_sales_order_line`, FK indexes on the fact, and `etl_control`.

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

Run it:

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/03_dw_ddl.sql
```

**Expected output** (first run): `NOTICE`s like `materialized view "mv_daily_store_category_sales" does not exist, skipping` and `table "fact_sales" does not exist, skipping`, then `DROP …`, `CREATE TABLE` / `CREATE INDEX` lines, and two `INSERT 0 1`. The script drops and recreates the warehouse, so it is safe to rerun (it empties all warehouse tables).

Check, in psql:

```sql
\dt public.*
SELECT * FROM dim_promotion;
SELECT * FROM etl_control;
```

**Expected output:**

```
             List of relations
 Schema |     Name      | Type  |  Owner
--------+---------------+-------+----------
 public | dim_customer  | table | postgres
 public | dim_date      | table | postgres
 public | dim_product   | table | postgres
 public | dim_promotion | table | postgres
 public | dim_store     | table | postgres
 public | etl_control   | table | postgres
 public | fact_sales    | table | postgres
(7 rows)

 promotion_key | promotion_id | promotion_name | promotion_type | discount_percent | start_date | end_date | is_active
---------------+--------------+----------------+----------------+------------------+------------+----------+-----------
            -1 |           -1 | No Promotion   | None           |                  |            |          | f
(1 row)

 table_name | high_water_mark
------------+---------------------
 fact_sales | 1970-01-01 00:00:00
(1 row)
```

✅ **Checkpoint:** 7 tables; the placeholder promotion and the `1970-01-01` high-water mark exist.

### Step 7 — Prove the constraints protect the facts

Foreign keys stop orphan facts — try inserting a sale for a date that isn't in `dim_date` (it is still empty):

```sql
INSERT INTO fact_sales (date_key, customer_key, product_key, store_key,
                        order_id, order_line_number, quantity, unit_price, unit_cost,
                        gross_revenue, net_revenue, gross_margin)
VALUES (20250101, 1, 1, 1, 1, 1, 1, 1.00, 0.50, 1.00, 1.00, 0.50);
```

**Expected output:**

```
ERROR:  insert or update on table "fact_sales" violates foreign key constraint "fact_sales_date_key_fkey"
DETAIL:  Key (date_key)=(20250101) is not present in table "dim_date".
```

This is why the load order is always **dimensions first, then facts** (Labs 05 → 06).

✅ **Checkpoint:** the insert is rejected.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `relation "dim_date" does not exist` in later labs | DDL not run, or run in another database | Rerun Step 6 against `freshmart` (`-d freshmart`). |
| `cannot drop table … because other objects depend on it` | Objects created by hand that depend on the warehouse | The script already uses `CASCADE`; drop your extra objects or rerun the script as given. |
| `\dt` shows the source tables but not `dim_*` | You're listing `source_oltp.*` | Use `\dt public.*`. |
| `promotion_key = -1` missing | DDL was edited / partially run | Rerun `03_dw_ddl.sql` with `ON_ERROR_STOP=1` and read the first error. |
| Rerunning the DDL wiped your loaded data | It drops and recreates tables by design | Reload with Lab 05 and Lab 06 scripts. |

## Key takeaways

- Grain first: one row per order line; every measure must be true at that grain.
- Star schema = one hop from fact to each denormalized dimension.
- Surrogate keys (`SERIAL`) enable SCD Type 2; inclusive effective dates + `1900-01-01` start + `9999-12-31` end + one-current partial index.
- `-1` placeholder rows instead of NULL FKs; `UNIQUE (order_id, order_line_number)` makes loads idempotent.

## Next

[Lab 05 — Loading the Dimensions](Lab_05_Loading_the_Dimensions.md): fill `dim_date`, the Type 1 dimensions and the SCD Type 2 dimensions.
