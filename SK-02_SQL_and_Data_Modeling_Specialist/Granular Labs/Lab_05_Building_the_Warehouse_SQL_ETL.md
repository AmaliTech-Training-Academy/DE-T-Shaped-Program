# Lab 05 — Building the Warehouse: SQL ETL

> **Scenario:** The star schema from Lab 04 is designed and empty. Now you build the machinery that fills it — and keeps filling it every night without human help, without loading anything twice, and without losing history. This is ETL in pure SQL.
>
> **Time:** 5–7 hours | **Difficulty:** Intermediate → Advanced | **Prerequisite:** Labs 01–04 (the `dw` schema must exist with all 5 tables)

---

## 1. Environment Setup

Verify the Lab 04 schema exists:

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema = 'dw' ORDER BY 1;
```

**Expected output:** `dim_customer, dim_date, dim_product, dim_store, fact_sales`.
**If missing:** rerun `04_star_schema.sql`. If the OLTP tables are missing, see Lab 02 Section 1.

Create the script and log folders:

```powershell
New-Item -ItemType File -Path C:\dataeng\sk02\sql\05_etl.sql -Force
New-Item -ItemType Directory -Path C:\dataeng\sk02\logs -Force
```

---

## 2. Business Context

**ETL** = Extract, Transform, Load: read from the source (OLTP tables), reshape (surrogate keys, SCD versions), write to the warehouse. FreshMart's warehouse will reload **every night at 2 a.m.**, unattended. That single fact drives every design decision in this lab:

- Nobody is watching → the load must **log** what it did.
- The scheduler may retry after a failure → the load must be **idempotent** (safe to run twice).
- 50,000 orders tonight, 50 million eventually → the load must be **incremental** (only new data), not full-reload.
- Customers change city/segment between runs → the load must apply **SCD Type 2** correctly.

**Who consumes it:** every dashboard and analyst downstream. **What happens if it fails badly:** the worst failure isn't a crash (visible, fixable) — it's a *silent double-load* that inflates revenue, or a *skipped SCD close* that attributes months of sales to the wrong segment. Both are careers-worth-of-trust expensive; both are prevented by patterns you build today.

---

## 3. Concept Explanation

### Idempotency

An operation is **idempotent** if running it twice has the same effect as running it once. Non-idempotent: `INSERT INTO fact SELECT ...` (run twice → doubles). Idempotent tools in PostgreSQL:

- `INSERT ... ON CONFLICT (key) DO NOTHING/DO UPDATE` — "upsert": the UNIQUE constraint on `order_line_id` (built in Lab 04, on purpose) turns duplicate loads into no-ops.
- `TRUNCATE` + reload — idempotent but *full* reload; fine for small dimensions, ruinous for big facts.

### Incremental loading and the high-water mark

A **high-water mark (HWM)** is a remembered value — "the latest `order_ts` I have loaded." Each run: read the mark, load only rows *after* it, advance the mark **in the same transaction**. Alternatives: change data capture (CDC — reading the DB's write-ahead log; the industrial-strength version, tool territory), or full reloads (simple, doesn't scale).

### The SCD Type 2 load pattern: close-then-insert

For each changed dimension row:
1. **Close** the current version: `UPDATE ... SET valid_to = today, is_current = false`.
2. **Insert** the new version: `valid_from = today, valid_to = '9999-12-31', is_current = true`.

The subtle killer is **change detection with NULLs**: `email <> new_email` is NULL (not true!) when either side is NULL, so changes to/from NULL go *undetected*. PostgreSQL's `IS DISTINCT FROM` is the NULL-safe not-equals — you met it in Lab 01; here is why it exists.

### Transactions

`BEGIN ... COMMIT` makes a group of statements **atomic** — all happen or none do. If the fact insert succeeds but advancing the HWM fails, you'd re-load the same rows forever. One transaction around both = impossible.

### ELT vs ETL

We're technically doing **ELT** — data is already *in* the database, we transform with SQL. This is the modern warehouse pattern (dbt, which you'll meet in SK-03, is exactly this industrialized). Classic ETL transforms *outside* the database first (needed when sources are files/APIs — your SK-01 Python skills).

---

## 4. Step-by-Step Implementation

### Step 1 — Populate dim_date (once, forever)

**What/why:** generate a decade of calendar rows with `generate_series` — the trick from Lab 01, now doing warehouse work.

```sql
INSERT INTO dw.dim_date
SELECT
    TO_CHAR(d, 'YYYYMMDD')::int          AS date_key,
    d::date                              AS full_date,
    EXTRACT(YEAR FROM d)::int            AS year,
    EXTRACT(QUARTER FROM d)::int         AS quarter,
    EXTRACT(MONTH FROM d)::int           AS month,
    TO_CHAR(d, 'FMMonth')                AS month_name,
    EXTRACT(DAY FROM d)::int             AS day_of_month,
    EXTRACT(ISODOW FROM d)::int          AS day_of_week,
    TO_CHAR(d, 'FMDay')                  AS day_name,
    EXTRACT(ISODOW FROM d) IN (6,7)      AS is_weekend
FROM generate_series('2023-01-01'::date, '2032-12-31'::date, '1 day') AS d
ON CONFLICT (date_key) DO NOTHING;
```

**Expected output:** `INSERT 0 3653` (10 years incl. leap days).
**Verify:** `SELECT COUNT(*), MIN(full_date), MAX(full_date) FROM dw.dim_date;` → 3653, 2023-01-01, 2032-12-31. Spot-check a known date: 2025-01-15 was a Wednesday.
**Common mistake:** forgetting `FM` in `TO_CHAR(d,'FMMonth')` — without it PostgreSQL pads names with spaces (`'May      '`) and string comparisons mysteriously fail later.
**Idempotency check (do this now):** run the whole INSERT again. **Expected:** `INSERT 0 0`. The `ON CONFLICT DO NOTHING` made it safe. This is the pattern of the day.

### Step 2 — Initial load of dim_store and dim_product

**What/why:** first-time (bulk) dimension load. Stores are Type 1 (simple); products get SCD2 columns initialized.

```sql
INSERT INTO dw.dim_store (store_id, store_name, region)
SELECT store_id, store_name, region FROM public.stores
ORDER BY store_id;

INSERT INTO dw.dim_product (product_id, product_name, category, unit_price, valid_from)
SELECT product_id, product_name, category, unit_price, CURRENT_DATE
FROM public.products
ORDER BY product_id;
```

**Expected output:** `INSERT 0 8`, `INSERT 0 25`.
**Verify:** `SELECT COUNT(*) FROM dw.dim_product WHERE is_current;` → 25.
**Common mistake:** running the initial load twice — unlike Step 1 there's no conflict target yet (dimension natural keys aren't unique by design!). If you doubled it: `TRUNCATE dw.dim_product RESTART IDENTITY CASCADE;` and reload. This is exactly why *incremental* dimension loads (Step 4) must check existence, and why Lab 06 adds a guard.

### Step 3 — Initial load of dim_customer

```sql
INSERT INTO dw.dim_customer (customer_id, full_name, email, city, segment, valid_from)
SELECT customer_id, full_name, email, city, segment, CURRENT_DATE
FROM public.customers
ORDER BY customer_id;
```

**Expected output:** `INSERT 0 2000`.
**Verify:** `SELECT COUNT(*) FROM dw.dim_customer WHERE is_current;` → 2000; and zero rows with `is_current = false`.

### Step 4 — The SCD Type 2 incremental load (the heart of this lab)

**What/why:** the reusable pattern that handles *changed* and *new* customers on every nightly run. Read it slowly — the CTE detects changes NULL-safely, then close-then-insert applies them atomically.

```sql
BEGIN;

-- 4a. Detect rows whose SCD2-tracked attributes changed
WITH changed AS (
    SELECT s.customer_id, s.full_name, s.email, s.city, s.segment
    FROM public.customers s
    JOIN dw.dim_customer d
      ON d.customer_id = s.customer_id AND d.is_current
    WHERE s.city    IS DISTINCT FROM d.city
       OR s.segment IS DISTINCT FROM d.segment
)
-- 4b. Close the current versions
UPDATE dw.dim_customer d
SET valid_to = CURRENT_DATE, is_current = FALSE
FROM changed c
WHERE d.customer_id = c.customer_id AND d.is_current;

-- 4c. Insert new versions for changed rows + brand-new customers
INSERT INTO dw.dim_customer (customer_id, full_name, email, city, segment, valid_from)
SELECT s.customer_id, s.full_name, s.email, s.city, s.segment, CURRENT_DATE
FROM public.customers s
LEFT JOIN dw.dim_customer d
       ON d.customer_id = s.customer_id AND d.is_current
WHERE d.customer_id IS NULL;            -- no current row: either just closed, or new

COMMIT;
```

**Test it — simulate real change:**

```sql
UPDATE public.customers SET city = 'Accra', segment = 'Premium' WHERE customer_id = 42;
-- now rerun the whole Step 4 block
SELECT customer_key, city, segment, valid_from, valid_to, is_current
FROM dw.dim_customer WHERE customer_id = 42 ORDER BY valid_from;
```

**Expected output:** two rows — the old version closed today with `is_current = false`, the new Accra/Premium version open-ended and current.
**Verify idempotency:** rerun Step 4 immediately. Zero updates, zero inserts (nothing is "changed" anymore). *Caveat you should notice:* if a customer changes twice in one day, this daily-grain pattern keeps only the last state that day — an accepted, documented limitation of date-grained SCD2.
**Common mistakes:**
- Using `<>` instead of `IS DISTINCT FROM` — NULL email changes silently missed.
- Doing INSERT before UPDATE — the LEFT JOIN then sees the new row as "current" and skips it, or worse, you close the row you just inserted.
- Forgetting `AND d.is_current` in joins — you'd compare against ancient versions.
**Troubleshooting:** ended up with two `is_current = true` rows for one customer? Your UPDATE didn't run (transaction rolled back?). Fix data: close the older one manually; fix cause: check for errors between BEGIN and COMMIT. Lab 06 adds a constraint making this state impossible.

### Step 5 — Create the ETL control table (high-water mark + audit)

**What/why:** the load needs memory (where did I stop?) and a diary (what did each run do?).

```sql
CREATE TABLE IF NOT EXISTS dw.etl_control (
    table_name      TEXT PRIMARY KEY,
    high_water_mark TIMESTAMPTZ NOT NULL
);
INSERT INTO dw.etl_control VALUES ('fact_sales', '1900-01-01')
ON CONFLICT (table_name) DO NOTHING;

CREATE TABLE IF NOT EXISTS dw.etl_run_log (
    run_id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name   TEXT NOT NULL,
    started_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at  TIMESTAMPTZ,
    rows_loaded  BIGINT,
    status       TEXT NOT NULL DEFAULT 'running'   -- running | success | failed
);
```

**Expected output:** two tables, one seed row.
**Verify:** `SELECT * FROM dw.etl_control;` shows fact_sales at 1900 — meaning "load everything" on first run.

### Step 6 — The incremental, idempotent fact load

**What/why:** everything converges here: HWM (only new orders), surrogate-key lookups (join dims on natural key *while current* — actually, on validity at order time), ON CONFLICT (idempotency), one transaction (atomicity), and logging.

```sql
BEGIN;

INSERT INTO dw.etl_run_log (table_name) VALUES ('fact_sales');

WITH mark AS (
    SELECT high_water_mark FROM dw.etl_control WHERE table_name = 'fact_sales'
),
new_lines AS (
    SELECT o.order_id, o.order_ts, o.store_id, o.customer_id,
           ol.order_line_id, ol.product_id, ol.quantity, ol.line_amount
    FROM public.orders o
    JOIN public.order_lines ol ON ol.order_id = o.order_id
    CROSS JOIN mark m
    WHERE o.status = 'completed'
      AND o.order_ts > m.high_water_mark
),
inserted AS (
    INSERT INTO dw.fact_sales
        (date_key, customer_key, product_key, store_key,
         order_id, order_line_id, quantity, line_amount)
    SELECT
        TO_CHAR(nl.order_ts, 'YYYYMMDD')::int,
        dc.customer_key, dp.product_key, ds.store_key,
        nl.order_id, nl.order_line_id, nl.quantity, nl.line_amount
    FROM new_lines nl
    JOIN dw.dim_customer dc ON dc.customer_id = nl.customer_id
                           AND nl.order_ts::date >= dc.valid_from
                           AND nl.order_ts::date <  dc.valid_to
    JOIN dw.dim_product  dp ON dp.product_id = nl.product_id AND dp.is_current
    JOIN dw.dim_store    ds ON ds.store_id   = nl.store_id
    ON CONFLICT (order_line_id) DO NOTHING
    RETURNING 1
)
UPDATE dw.etl_control
SET high_water_mark = COALESCE(
        (SELECT MAX(order_ts) FROM public.orders WHERE status = 'completed'),
        high_water_mark)
WHERE table_name = 'fact_sales';

UPDATE dw.etl_run_log
SET finished_at = now(), status = 'success',
    rows_loaded = (SELECT COUNT(*) FROM dw.fact_sales)
WHERE run_id = (SELECT MAX(run_id) FROM dw.etl_run_log);

COMMIT;
```

**Expected output:** first run loads ~140k fact rows (completed orders' lines).
**Verify (three ways):**
1. `SELECT COUNT(*) FROM dw.fact_sales;` ≈ completed order lines in source.
2. Reconciliation: `SELECT SUM(line_amount) FROM dw.fact_sales;` equals source sum for completed orders — the query from Lab 02 Step 2's verification. **Source-to-target reconciliation is the single most important ETL test.**
3. `SELECT * FROM dw.etl_control;` — HWM advanced to the max order timestamp.
**Idempotency proof:** run the whole block again → the HWM filter finds nothing; even if it did, ON CONFLICT drops duplicates. Fact count unchanged.
**Common mistakes:**
- Joining `dim_customer` on `is_current` instead of validity range — facts would point at *today's* version, destroying the "segment at time of sale" property that justified SCD2.
- Setting HWM to `now()` instead of `MAX(order_ts)` — clock skew or in-flight transactions could skip rows.
**Troubleshooting:** `null value in column "customer_key"` → an order references a customer with no dimension row covering its date (a **late-arriving dimension** — the customer changed before their history was loaded, or initial `valid_from` postdates old orders). Quick fix for the lab: `UPDATE dw.dim_customer SET valid_from = '2023-01-01' WHERE valid_from = CURRENT_DATE AND is_current;` — and remember this failure mode: it's a core capstone scenario.

### Step 7 — Simulate the next nightly run

**What/why:** prove incrementality end-to-end.

```sql
-- New business arrives:
INSERT INTO public.orders (customer_id, store_id, order_ts, status)
VALUES (42, 3, now(), 'completed');
INSERT INTO public.order_lines (order_id, product_id, quantity, line_amount)
SELECT MAX(order_id), 7, 2, 19.98 FROM public.orders;
```

Rerun Step 6. **Expected:** exactly 1 new fact row (`SELECT COUNT(*)` went up by 1), pointing at customer 42's *new* (Accra/Premium) dimension version — verify:

```sql
SELECT f.order_id, c.city, c.segment
FROM dw.fact_sales f JOIN dw.dim_customer c USING (customer_key)
WHERE f.order_id = (SELECT MAX(order_id) FROM public.orders);
```

**Expected output:** `Accra | Premium` — while customer 42's *old* facts still show Kumasi/Standard context. SCD2, working end to end. Take a moment; this is the hardest pattern in the module and you just built it.

Save everything to `05_etl.sql`.

---

## 5. Production Engineering Practices

1. **Idempotency as a constraint, not a convention.** The UNIQUE on `order_line_id` means even a buggy re-run *cannot* double-load. *Failure story:* a scheduler retried a "failed" job that had actually committed; without a key constraint, that warehouse showed +18% phantom revenue for three days before anyone questioned it.
2. **HWM and data in one transaction.** Advance-the-mark and load-the-rows must commit together. Separated, a crash between them either re-loads (duplicates) or skips (gaps).
3. **Audit logging from day one.** `etl_run_log` costs three statements and answers "did last night run? how long? how many rows?" — the first three questions in every incident. In Lab 06 it feeds monitoring.
4. **Reconciliation checks after every load.** Source sum = target sum, or the run is a failure even if it "succeeded." Automate it (Lab 06).
5. **Script organization.** In real deployments these blocks become separate numbered files (`10_load_dims.sql`, `20_load_facts.sql`) run by a scheduler via `psql -f`, with `ON_ERROR_STOP=1` so failures halt the chain: `psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f 05_etl.sql`. Try that command now from PowerShell — it's how Task Scheduler would run your nightly load.

---

## 6. Reflection

**What you learned:** dim_date generation, initial vs incremental dimension loads, the SCD2 close-then-insert pattern with NULL-safe change detection, high-water-mark incremental fact loads with surrogate key lookups against validity ranges, transactional atomicity, ON CONFLICT idempotency, and run logging.

**Why it matters:** this lab *is* the job. Everything else in the module supports what you built here.

### Interview questions

1. **What makes a data load idempotent, and why does it matter?** *Same result no matter how many times it runs — via upserts on a natural key, or delete-then-insert of the affected window. Matters because schedulers retry and humans re-run; without it, duplicates.*
2. **Explain a high-water-mark incremental load and its failure modes.** *Remember the max loaded timestamp, load only newer rows, advance the mark in the same transaction. Failure modes: late-arriving source rows behind the mark (missed), using wall-clock now() as the mark (skips in-flight data).*
3. **Walk me through an SCD Type 2 load.** *Detect changes on tracked columns NULL-safely (IS DISTINCT FROM), UPDATE to close current versions (set valid_to, is_current=false), INSERT new versions, all in one transaction. Insert brand-new keys as fresh current rows.*
4. **Why must fact loads join dimensions on validity ranges rather than is_current?** *The fact must reference the dimension version current at event time, or history is silently rewritten to today's attributes.*
5. **What is a late-arriving dimension and how do you handle it?** *A fact arrives before its dimension row. Options: fail the row into a dead-letter table for retry, insert a placeholder ("unknown") dimension row and correct it later, or hold the fact. Choice is a documented business decision.*
6. **How do you verify a load succeeded beyond 'no error was thrown'?** *Reconciliation: source vs target row counts and sums; duplicate checks on the grain key; referential checks; freshness check on the audit log.*
7. **Why wrap the fact insert and HWM update in one transaction?** *Atomicity — a crash between them causes duplicates or gaps. Together, either both happen or neither.*
8. **ETL vs ELT?** *ETL transforms before loading (external tools; needed for file/API sources). ELT lands raw data then transforms in-database with SQL — the modern warehouse pattern (dbt).*

**Common trap:** "Your load is idempotent because you check the HWM, right?" No — HWM gives *incrementality*; idempotency needs the conflict-safe insert too. A re-run after a partial failure can straddle the mark; they're separate guarantees that compose.

**Next:** [Lab 06 — Data Quality & Production Hardening](Lab_06_Data_Quality_Production_Hardening.md): the warehouse works — now make it trustworthy and fast under fire.
