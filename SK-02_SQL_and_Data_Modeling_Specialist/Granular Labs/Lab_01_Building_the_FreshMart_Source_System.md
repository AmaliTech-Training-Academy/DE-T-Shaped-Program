# Lab 01 — Building the FreshMart Source System

> **Goal:** Create FreshMart's operational (OLTP) database in schema `source_oltp` — tables, keys and constraints — and fill it with deterministic sample data.
> **Time:** 120–150 min | **Difficulty:** Beginner | **Prerequisites:** Lab 00
> **Guided lab coverage:** PART 0 (Steps 0.1–0.2) and PART 1 Step 1.1 of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Explain **OLTP**, **3NF** (normalization), **schema**, **table**, **row**, **column** and **data type**.
- Create a schema and tables with `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, `UNIQUE`, `CHECK` and `DEFAULT`, and read the error each constraint raises.
- Choose correct PostgreSQL types: `SERIAL`, `INT`, `SMALLINT`, `DECIMAL(10,2)` for money, `VARCHAR(n)`, `CHAR(n)`, `DATE`, `TIMESTAMP`, `BOOLEAN`.
- Use composite primary keys and partial unique indexes.
- Generate rows with `generate_series()` and make "random" data repeatable with a deterministic function.
- Profile a database with `pg_stat_user_tables` and `pg_size_pretty()` (guided Step 1.1).

**Key terms**

| Term | Meaning |
|---|---|
| **OLTP** (Online Transaction Processing) | A database built for many small, fast writes — ring up a sale, register a customer. Its opposite, **OLAP**, is built for big analytical reads (Labs 04–08). |
| **3NF** (Third Normal Form) | A design where every fact is stored exactly once: a region name lives in one row of `store_regions`; stores point to it by id. Great for writes, painful for reports (many joins). |
| **Schema** | A named folder of tables inside a database. Source tables go in `source_oltp`; the warehouse (Lab 04) goes in the default schema `public`. |
| **Primary key (PK)** | Column(s) that uniquely identify each row; never NULL. |
| **Foreign key (FK)** | A column that must match an existing PK in another table — "every order belongs to a real customer". |

## Steps

Open an interactive session for Steps 1–4 and 6 (`docker exec -it sk02-postgres psql -U postgres -d freshmart`).

### Step 1 — Create (and remove) a schema

Experiments in this lab happen in a throw-away schema called `demo`, so they never disturb the real source tables.

```sql
CREATE SCHEMA demo;
\dn
```

**Expected output:**

```
CREATE SCHEMA
      List of schemas
  Name  |       Owner
--------+-------------------
 demo   | postgres
 public | pg_database_owner
(2 rows)
```

`public` exists in every database; it is where tables go when you don't name a schema. From now on always write `schema.table` (for example `source_oltp.orders`) so it is obvious which one you mean.

✅ **Checkpoint:** `\dn` lists `demo`.

### Step 2 — Tables, data types and constraints

The source's top-level store table looks like this. Create it in `demo` and try to break it:

```sql
CREATE TABLE demo.store_regions (
    region_id    SERIAL       PRIMARY KEY,     -- SERIAL = auto-numbered integer 1, 2, 3...
    region_name  VARCHAR(50)  NOT NULL UNIQUE  -- required, and no duplicates
);

INSERT INTO demo.store_regions (region_name) VALUES ('Northeast'), ('West');
SELECT * FROM demo.store_regions;

INSERT INTO demo.store_regions (region_name) VALUES (NULL);    -- breaks NOT NULL
INSERT INTO demo.store_regions (region_name) VALUES ('West');  -- breaks UNIQUE
```

**Expected output:**

```
CREATE TABLE
INSERT 0 2
 region_id | region_name
-----------+-------------
         1 | Northeast
         2 | West
(2 rows)

ERROR:  null value in column "region_name" of relation "store_regions" violates not-null constraint
DETAIL:  Failing row contains (3, null).
ERROR:  duplicate key value violates unique constraint "store_regions_region_name_key"
DETAIL:  Key (region_name)=(West) already exists.
```

Notice the failed inserts still used up sequence numbers (the `3` in the first DETAIL) — sequences never go backwards. That is why the real source tables in Step 5 are only ever filled once, by the seed script.

**Types you will meet in the source:**

| Type | Use | Example |
|---|---|---|
| `SERIAL` / `INT` / `SMALLINT` | Whole numbers (SERIAL auto-numbers) | `order_id`, `line_number` |
| `DECIMAL(10,2)` | **Exact** money — never `FLOAT` (0.1 + 0.2 ≠ 0.3 in floating point) | `unit_price` |
| `VARCHAR(n)` / `CHAR(2)` | Text up to n chars / exactly 2 chars | `store_name` / `country` |
| `DATE` / `TIMESTAMP` | Calendar day / day + time | `order_date` / `updated_at` |
| `BOOLEAN` | `TRUE` / `FALSE` / `NULL` | `is_active` |

✅ **Checkpoint:** you saw both errors and can say which constraint caused each.

### Step 3 — Foreign keys and CHECK constraints

```sql
CREATE TABLE demo.store_districts (
    district_id   SERIAL       PRIMARY KEY,
    district_name VARCHAR(100) NOT NULL,
    region_id     INT          NOT NULL REFERENCES demo.store_regions(region_id)
);
INSERT INTO demo.store_districts (district_name, region_id) VALUES ('Boston District', 1);
INSERT INTO demo.store_districts (district_name, region_id) VALUES ('Nowhere District', 99);

CREATE TABLE demo.products (
    product_id  SERIAL        PRIMARY KEY,
    list_price  DECIMAL(10,2) NOT NULL CHECK (list_price >= 0)
);
INSERT INTO demo.products (list_price) VALUES (-5);
```

**Expected output:**

```
CREATE TABLE
INSERT 0 1
ERROR:  insert or update on table "store_districts" violates foreign key constraint "store_districts_region_id_fkey"
DETAIL:  Key (region_id)=(99) is not present in table "store_regions".
CREATE TABLE
ERROR:  new row for relation "products" violates check constraint "products_list_price_check"
DETAIL:  Failing row contains (1, -5.00).
```

Because of foreign keys, **parents must be created and filled before children** (regions → districts → stores; customers → orders → order_items) and dropped in the reverse order.

✅ **Checkpoint:** you can explain why `region_id = 99` was rejected.

### Step 4 — Composite keys and partial unique indexes

Two source tables need special keys:

```sql
-- One order has many lines: the PK is the PAIR (order_id, line_number)
CREATE TABLE demo.order_items (
    order_id    INT      NOT NULL,
    line_number SMALLINT NOT NULL,
    quantity    INT      NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, line_number)
);
INSERT INTO demo.order_items VALUES (1, 1, 2), (1, 2, 1), (2, 1, 5);
INSERT INTO demo.order_items VALUES (1, 2, 9);

-- A customer can have many addresses but only ONE primary address
CREATE TABLE demo.customer_addresses (
    address_id  SERIAL  PRIMARY KEY,
    customer_id INT     NOT NULL,
    is_primary  BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE UNIQUE INDEX uq_demo_primary ON demo.customer_addresses(customer_id) WHERE is_primary;
INSERT INTO demo.customer_addresses (customer_id, is_primary) VALUES (7, TRUE), (7, FALSE), (7, FALSE);
INSERT INTO demo.customer_addresses (customer_id) VALUES (7);
```

**Expected output:**

```
CREATE TABLE
INSERT 0 3
ERROR:  duplicate key value violates unique constraint "order_items_pkey"
DETAIL:  Key (order_id, line_number)=(1, 2) already exists.
CREATE TABLE
CREATE INDEX
INSERT 0 3
ERROR:  duplicate key value violates unique constraint "uq_demo_primary"
DETAIL:  Key (customer_id)=(7) already exists.
```

A **partial index** (`... WHERE is_primary`) only covers the rows matching its condition — here it enforces "at most one primary address" while allowing any number of non-primary ones. You'll reuse this trick in Lab 04 for "at most one *current* version per customer".

Clean up the playground:

```sql
DROP SCHEMA demo CASCADE;
```

**Expected output:**

```
NOTICE:  drop cascades to 5 other objects
DETAIL:  drop cascades to table demo.store_regions
drop cascades to table demo.store_districts
drop cascades to table demo.products
drop cascades to table demo.order_items
drop cascades to table demo.customer_addresses
DROP SCHEMA
```

✅ **Checkpoint:** `\dn` no longer lists `demo`.

### Step 5 — Create the real source schema

Here is the full FreshMart source — 15 tables. Read it top to bottom: every concept is one you just practised. Note the **snowflaked hierarchies** (product → subcategory → category; store → district → region) and `updated_at` columns (they drive incremental loading in Lab 06). The last object is a helper function you'll use in Step 6.

Save it as `C:\DataEngineering\sk02\sql\01_source_oltp_ddl.sql` — it is identical to the guided lab's Step 0.1:

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

Run it (from PowerShell):

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/01_source_oltp_ddl.sql
```

**Expected output** (first run):

```
psql:/sql/01_source_oltp_ddl.sql:6: NOTICE:  schema "source_oltp" does not exist, skipping
DROP SCHEMA
CREATE SCHEMA
CREATE TABLE
... (15 × CREATE TABLE in total, plus one CREATE INDEX)
CREATE FUNCTION
```

The script starts with `DROP SCHEMA IF EXISTS source_oltp CASCADE`, so it is **rerunnable**: running it again rebuilds everything from scratch (and deletes any data — you will reseed in Step 7).

Then check, in psql:

```sql
\dt source_oltp.*
```

**Expected output:**

```
                  List of relations
   Schema    |        Name        | Type  |  Owner
-------------+--------------------+-------+----------
 source_oltp | customer_addresses | table | postgres
 source_oltp | customer_segments  | table | postgres
 source_oltp | customers          | table | postgres
 source_oltp | employees          | table | postgres
 source_oltp | order_items        | table | postgres
 source_oltp | orders             | table | postgres
 source_oltp | product_categories | table | postgres
 source_oltp | product_subcats    | table | postgres
 source_oltp | products           | table | postgres
 source_oltp | promotion_products | table | postgres
 source_oltp | promotions         | table | postgres
 source_oltp | store_districts    | table | postgres
 source_oltp | store_regions      | table | postgres
 source_oltp | stores             | table | postgres
 source_oltp | suppliers          | table | postgres
(15 rows)
```

✅ **Checkpoint:** 15 tables listed; `\d source_oltp.order_items` shows `PRIMARY KEY, btree (order_id, line_number)`.

### Step 6 — Generate rows: `generate_series` and deterministic "randomness"

`generate_series(start, stop)` returns one row per number — a data factory:

```sql
SELECT gs AS n, 'Supplier ' || lpad(gs::text, 2, '0') AS supplier_name
FROM generate_series(1, 3) AS gs;
```

**Expected output:**

```
 n | supplier_name
---+---------------
 1 | Supplier 01
 2 | Supplier 02
 3 | Supplier 03
(3 rows)
```

`random()` would give every trainee different data — and then nobody's numbers would match the lab. `source_oltp.pseudo_random(seed, salt)` hashes its inputs with `md5`, so it always returns the same number between 0 and 1 for the same inputs:

```sql
SELECT source_oltp.pseudo_random(1, 'demo')  AS a,
       source_oltp.pseudo_random(42, 'demo') AS b,
       1 + floor(source_oltp.pseudo_random(1, 'demo') * 60)::int AS store_between_1_and_60;
```

**Expected output** (identical on every machine):

```
           a            |           b            | store_between_1_and_60
------------------------+------------------------+------------------------
 0.84901437698863446712 | 0.02241785428486764431 |                     51
(1 row)
```

`1 + floor(x * N)` turns a number in [0, 1) into a whole number from 1 to N — the seed uses this pattern everywhere (stores, products, quantities). A different **salt** gives an independent number for the same seed, so for example an order's store and its date don't move in lock-step.

✅ **Checkpoint:** your three values match exactly.

### Step 7 — Seed the source with deterministic data

Save as `C:\DataEngineering\sk02\sql\02_source_oltp_seed.sql` (identical to guided Step 0.2). Look for these patterns as you read: `generate_series` row factories, `CASE` for categories ("15% have no email"), `CROSS JOIN LATERAL (SELECT ...)` to compute a helper value per row, and the insert **order** (parents before children).

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

Run it (1–2 minutes):

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/02_source_oltp_seed.sql
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

Running the seed a second time fails with a duplicate-key error (the data is already there) — to start over, rerun `01_source_oltp_ddl.sql` first, then the seed.

✅ **Checkpoint:** the last two inserts report exactly `100000` and `300235`.

### Step 8 — Profile the source (guided Step 1.1)

`pg_stat_user_tables` is a built-in **system view** with one row per table and live statistics; `pg_total_relation_size()` returns bytes (table + indexes) and `pg_size_pretty()` makes them human-readable.

```sql
SELECT
    relname          AS table_name,
    n_live_tup       AS estimated_row_count,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE schemaname = 'source_oltp'
ORDER BY n_live_tup DESC;
```

**Expected output** (sizes may differ slightly; row counts must match):

```
     table_name     | estimated_row_count | total_size
--------------------+---------------------+------------
 order_items        |              300235 | ...
 orders             |              100000 | ...
 customer_addresses |               22000 | ...
 customers          |               20000 | ...
 promotion_products |                2400 | ...
 products           |                1200 | ...
 employees          |                 300 | ...
 stores             |                  60 | ...
 promotions         |                  40 | ...
 product_subcats    |                  24 | ...
 suppliers          |                  20 | ...
 store_districts    |                  12 | ...
 product_categories |                   6 | ...
 store_regions      |                   4 | ...
 customer_segments  |                   3 | ...
(15 rows)
```

Now confirm the business mix with a real count:

```sql
SELECT status, COUNT(*) AS orders
FROM source_oltp.orders
GROUP BY status
ORDER BY status;
```

**Expected output:**

```
  status   | orders
-----------+--------
 cancelled |   4026
 completed |  93986
 returned  |   1988
(3 rows)
```

✅ **Checkpoint:** both outputs match. You have now done everything in guided lab PART 0 and Step 1.1.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: schema "demo" already exists` | You reran Step 1 | `DROP SCHEMA demo CASCADE;` then continue. |
| `psql: error: /sql/01_source_oltp_ddl.sql: No such file or directory` | File not saved in `C:\DataEngineering\sk02\sql\` or misnamed | `Get-ChildItem C:\DataEngineering\sk02\sql`; fix the name/location. |
| Seed fails with `duplicate key value violates unique constraint "store_regions_region_name_key"` | Seed was run twice | Rerun `01_source_oltp_ddl.sql`, then `02_source_oltp_seed.sql`. |
| Seed fails with `insert or update ... violates foreign key constraint` | Tables were changed or a statement was reordered | Use the scripts exactly as given; parents must be inserted before children. |
| `estimated_row_count` is 0 or lower than expected | Statistics not flushed yet | Wait a few seconds and rerun, or run `ANALYZE;`. |
| `function source_oltp.pseudo_random(integer, unknown) does not exist` | DDL script didn't finish | Rerun Step 5 and check for errors (`ON_ERROR_STOP=1` stops at the first one). |

## Key takeaways

- The source is a 3NF OLTP system in schema `source_oltp`: 15 tables, snowflaked store and product hierarchies, `updated_at` audit columns.
- Constraints (PK, FK, NOT NULL, UNIQUE, CHECK, partial unique indexes) are the database enforcing business rules for you.
- `generate_series` + a deterministic hash function = realistic, repeatable test data.
- `pg_stat_user_tables` + `pg_size_pretty` are the first thing to run on an unfamiliar database.

## Next

[Lab 02 — SQL Querying Foundations](Lab_02_SQL_Querying_Foundations.md): query the source you just built — filtering, NULLs, joins across the hierarchies and aggregation.
