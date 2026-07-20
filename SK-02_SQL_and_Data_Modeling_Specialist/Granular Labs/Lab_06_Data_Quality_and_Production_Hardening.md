# Lab 06 — Data Quality & Production Hardening

> **Module:** SK-02 SQL & Data Modeling Specialist
> **Estimated time:** 4–5 hours
> **Prerequisites:** Lab 05 — the FreshMart warehouse loads dimensions (SCD2) and facts incrementally.

---

## 1. Environment Setup

Nothing new to install. Verify your Lab 05 state is intact:

```powershell
psql -U postgres -d freshmart -c "SELECT count(*) FROM dw.fact_sales; SELECT count(*) FROM dw.dim_customer;"
```

**Expected output:** non-zero counts for both. If the tables are missing, re-run your Lab 05 ETL script first.

Create a folder for this lab's scripts:

```powershell
mkdir C:\sk02\lab06 -Force
```

**Common problems:** `psql` not recognized → PostgreSQL `bin` folder not on PATH (Lab 00 troubleshooting table). Password prompts every time → set the `PGPASSWORD` env var for the session or use a `pgpass.conf` file (never hardcode it in scripts).

---

## 2. Business Context

Your FreshMart warehouse now reloads nightly at 2 a.m., unattended. Tonight, one of these *will* eventually happen:

- The POS system exports a sale with a `customer_id` that doesn't exist (a race condition in their system).
- A price update makes `unit_price` negative for 14 rows.
- The export job runs twice, delivering the same day's sales twice.
- The load fails halfway, and the morning team needs to know *what happened and what to do* — before the 8 a.m. replenishment run that reads `fact_sales`.

**Who cares?** Buyers ordering stock from `fact_sales`, finance reconciling revenue, and the on-call engineer reading your audit trail at 6 a.m. If bad rows load silently, FreshMart orders the wrong stock — real money. If the load fails without a trail, diagnosis takes hours instead of minutes.

**Where you'll see this in industry:** every warehouse team maintains exactly these four defenses — database constraints, quality checks with dead-letter routing, run auditing, and idempotent loads. In vendor evaluations and job interviews, "how do you stop bad data reaching the fact table?" is a standard question, and this lab is the answer.

---

## 3. Concept Explanation

### 3.1 Constraints: the database as last line of defense

A **constraint** is a rule the database itself enforces — no code path can sneak around it:

| Constraint | Enforces | FreshMart example |
|---|---|---|
| `NOT NULL` | Value must exist | every fact row must have a date key |
| `CHECK` | Expression must be true | `quantity > 0`, `unit_price >= 0` |
| `UNIQUE` | No duplicates | one current row per customer in SCD2 |
| `FOREIGN KEY` | Value must exist in parent table | `fact_sales.customer_key` → `dim_customer` |

**Why both constraints *and* quality checks?** Constraints are binary and per-row: they reject with an error and (if unhandled) abort the load. Quality checks are programmable: they can route bad rows aside, count them, and let the good 99.9% load. Production systems use **constraints as the safety net** and **checks as the triage system** in front of it.

**Trade-off to know:** foreign keys add write overhead, and some high-volume warehouses drop them in favor of check-based validation (Redshift and Snowflake don't even enforce them). In PostgreSQL at FreshMart's volume, keep them on — integrity is worth microseconds.

### 3.2 Dead-letter tables

The SQL version of SK-01's dead-letter files: a table holding rows that failed validation, with *when*, *which rule*, and *the original row* (stored as JSONB so any shape fits). Nothing is deleted; someone reviews the dead letters; recurring patterns become upstream fixes.

### 3.3 The audit table

An **audit table** records one row per ETL run: start/end time, rows read/loaded/rejected per step, status. It answers the three operational questions instantly: *Did last night run? How long did it take vs normal? Where did rows go?* It's also your monitoring hook — a query over the audit table ("no successful run in 26 hours" or "rejects > 1%") is what an alerting tool would poll.

### 3.4 Idempotent SQL loads

Same principle as SK-01, new tools. The SQL idioms:

- **Delete-then-insert by scope**: `DELETE FROM fact_sales WHERE date_key = :run_date` then insert that date's rows — re-running a date is safe.
- **`INSERT ... ON CONFLICT DO NOTHING/UPDATE`** ("upsert"): natural-key collisions become no-ops or updates instead of duplicates.
- **High-water mark** (Lab 05): only process rows newer than the last successful load — but pair it with one of the above, because *the same batch can still be delivered twice*.
- **Transactions**: wrap each load step in `BEGIN ... COMMIT` so a mid-step failure rolls back cleanly — no half-loaded state.

---

## 4. Step-by-Step Implementation

### Step 1 — Harden the schema with constraints

**What:** Create `C:\sk02\lab06\01_constraints.sql`:

```sql
-- Facts: business-rule checks
ALTER TABLE dw.fact_sales
    ADD CONSTRAINT chk_quantity_positive CHECK (quantity > 0),
    ADD CONSTRAINT chk_unit_price_non_negative CHECK (unit_price >= 0);

-- SCD2 hygiene: exactly one current row per business key
CREATE UNIQUE INDEX IF NOT EXISTS uq_dim_customer_current
    ON dw.dim_customer (customer_id) WHERE is_current;

-- Referential integrity (if not already present from Lab 04)
ALTER TABLE dw.fact_sales
    ADD CONSTRAINT fk_sales_customer FOREIGN KEY (customer_key)
        REFERENCES dw.dim_customer (customer_key);
```

Run it:

```powershell
psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f C:\sk02\lab06\01_constraints.sql
```

**Why:** the partial unique index (`WHERE is_current`) is the classic SCD2 guard — it makes "two current versions of the same customer" *impossible*, which is the most damaging SCD2 corruption (every fact join doubles).

**Expected output:** `ALTER TABLE` / `CREATE INDEX` confirmations. If `chk_quantity_positive` fails with `check constraint ... is violated by some row`, congratulations — you already have bad data. Find it (`SELECT * FROM dw.fact_sales WHERE quantity <= 0;`), decide (fix or delete to a quarantine table), then re-apply.

**Verify the constraint actually fires:**

```sql
INSERT INTO dw.fact_sales (date_key, customer_key, product_key, quantity, unit_price, line_amount)
VALUES (20260101, 1, 1, -5, 10, -50);
-- ERROR:  new row ... violates check constraint "chk_quantity_positive"
```

An untested constraint is a hope, not a control.

**Common mistake:** adding constraints *before* profiling existing data — the ALTER fails and you assume the syntax is wrong. It's the data. Always profile first (Lab 03 skills).

### Step 2 — Build the dead-letter table and validating load

**What:** Create `02_dead_letter.sql`:

```sql
CREATE TABLE IF NOT EXISTS etl.dead_letter (
    dead_letter_id  BIGSERIAL PRIMARY KEY,
    rejected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    source_table    TEXT NOT NULL,
    rule_violated   TEXT NOT NULL,
    raw_row         JSONB NOT NULL
);

-- Validating load: good rows to staging-clean, bad rows to dead_letter
WITH classified AS (
    SELECT s.*,
        CASE
            WHEN s.quantity IS NULL OR s.quantity <= 0        THEN 'quantity_not_positive'
            WHEN s.unit_price < 0                             THEN 'negative_unit_price'
            WHEN c.customer_key IS NULL                       THEN 'unknown_customer'
            ELSE NULL
        END AS violation
    FROM staging.sales s
    LEFT JOIN dw.dim_customer c
           ON c.customer_id = s.customer_id AND c.is_current
),
rejected AS (
    INSERT INTO etl.dead_letter (source_table, rule_violated, raw_row)
    SELECT 'staging.sales', violation, to_jsonb(classified)
    FROM classified WHERE violation IS NOT NULL
    RETURNING 1
)
INSERT INTO staging.sales_clean
SELECT * FROM classified WHERE violation IS NULL;
```

(Adapt column/table names to your Lab 05 staging structures.)

**Why this shape:** one `WITH` statement classifies every row exactly once, routes rejects with their reason *and full original row* (JSONB), and passes clean rows on — atomically, in one transaction. No row can be lost between the two destinations.

**Verify:** insert a staged row with `quantity = -1`, run the load, and check:

```sql
SELECT rule_violated, count(*) FROM etl.dead_letter GROUP BY 1;
```

**Common mistakes:**
- Validating with several separate `DELETE ... WHERE bad` statements — rows matching two rules get double-handled, and you lose the reason. Classify once with `CASE`.
- Storing only the bad row's ID, not the row itself. Upstream data changes; JSONB snapshot preserves the evidence.

### Step 3 — Create the audit table and instrument the ETL

**What:** Create `03_audit.sql`:

```sql
CREATE TABLE IF NOT EXISTS etl.run_audit (
    run_id        BIGSERIAL PRIMARY KEY,
    step_name     TEXT NOT NULL,
    started_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at   TIMESTAMPTZ,
    rows_read     BIGINT,
    rows_loaded   BIGINT,
    rows_rejected BIGINT,
    status        TEXT NOT NULL DEFAULT 'running'
                  CHECK (status IN ('running','success','failed'))
);
```

Then bracket each ETL step in your Lab 05 script:

```sql
-- start of step
INSERT INTO etl.run_audit (step_name) VALUES ('load_fact_sales');

-- ... the load itself, capturing counts with GET DIAGNOSTICS in a DO block,
--     or simpler: count from the tables after the step ...

-- end of step
UPDATE etl.run_audit
SET finished_at = now(),
    rows_read     = (SELECT count(*) FROM staging.sales),
    rows_loaded   = (SELECT count(*) FROM staging.sales_clean),
    rows_rejected = (SELECT count(*) FROM etl.dead_letter
                     WHERE rejected_at >= (SELECT started_at FROM etl.run_audit
                                           WHERE run_id = currval('etl.run_audit_run_id_seq'))),
    status = 'success'
WHERE run_id = currval('etl.run_audit_run_id_seq');
```

**Why:** `rows_read = rows_loaded + rows_rejected` is your reconciliation identity — if it doesn't hold, rows vanished, and that's a P1 bug. The audit table makes the identity checkable *per run, forever*.

**Verify with the monitoring queries an operator would run:**

```sql
-- Did last night succeed, and was it normal?
SELECT * FROM etl.run_audit ORDER BY run_id DESC LIMIT 5;

-- Alert condition 1: no successful load in 26 hours
SELECT count(*) = 0 AS should_alert
FROM etl.run_audit
WHERE step_name = 'load_fact_sales' AND status = 'success'
  AND finished_at > now() - interval '26 hours';

-- Alert condition 2: reject rate above 1%
SELECT rows_rejected::numeric / NULLIF(rows_read,0) > 0.01 AS should_alert
FROM etl.run_audit ORDER BY run_id DESC LIMIT 1;
```

These two queries **are** your monitoring strategy — in production a scheduler or observability tool simply runs them and emails on `true`. Write that sentence in your project docs; it's the honest local-only answer to "where's your monitoring?"

### Step 4 — Prove idempotency

**What:** make the fact load re-runnable, then prove it. If your Lab 05 load uses a high-water mark, add the delete-by-scope guard:

```sql
BEGIN;

DELETE FROM dw.fact_sales
WHERE date_key IN (SELECT DISTINCT date_key_of(s.sale_ts) FROM staging.sales_clean s);

INSERT INTO dw.fact_sales (...)
SELECT ... FROM staging.sales_clean ...;

COMMIT;
```

**The proof ritual:**

```powershell
psql -U postgres -d freshmart -c "SELECT count(*), sum(line_amount) FROM dw.fact_sales;"
psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f C:\sk02\lab05\05_etl.sql
psql -U postgres -d freshmart -c "SELECT count(*), sum(line_amount) FROM dw.fact_sales;"
```

**Expected output:** identical count *and* identical sum before and after the re-run. Count alone can lie (equal counts of different rows); count + amount is a solid cheap fingerprint.

**Why the transaction matters:** if the insert fails after the delete, `ROLLBACK` restores the deleted rows. Without `BEGIN/COMMIT`, a mid-step crash leaves the date's data *gone* — the worst possible outcome of a "safety" mechanism.

**Common mistake:** deleting by `WHERE date_key = today` while the staged batch actually contains *yesterday's late rows* too. Scope the delete by **the dates present in the batch** (as above), not by the calendar.

### Step 5 — The quality report

**What:** Create `04_quality_report.sql` — checks that run *after* each load, reporting pass/fail like your SK-01 validator:

```sql
WITH checks AS (
    SELECT 'fact_sales.no_orphan_customers' AS check_name,
           count(*) AS failed
    FROM dw.fact_sales f
    LEFT JOIN dw.dim_customer c ON c.customer_key = f.customer_key
    WHERE c.customer_key IS NULL
    UNION ALL
    SELECT 'dim_customer.one_current_per_id',
           count(*) FROM (
               SELECT customer_id FROM dw.dim_customer
               WHERE is_current GROUP BY customer_id HAVING count(*) > 1) d
    UNION ALL
    SELECT 'fact_sales.amount_consistency',
           count(*) FROM dw.fact_sales
    WHERE round(quantity * unit_price, 2) <> round(line_amount, 2)
    UNION ALL
    SELECT 'fact_sales.no_future_dates',
           count(*) FROM dw.fact_sales
    WHERE date_key > to_char(current_date, 'YYYYMMDD')::int
)
SELECT check_name, failed,
       CASE WHEN failed = 0 THEN 'PASS' ELSE 'FAIL' END AS status
FROM checks
ORDER BY failed DESC;
```

**Why these four:** each targets a *different corruption mode* — broken references, SCD2 duplication, arithmetic drift, and time-travel data. Notice the philosophical difference from constraints: these validate **relationships and invariants across rows**, which single-row constraints can't express.

**Verify:** run it; all four should PASS. Then break one on purpose (flip an `is_current` flag on a duplicate row — in a transaction you ROLLBACK afterward!) and watch it FAIL. Alarm tested.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Constraints as the last line of defense** | No code path can bypass the database itself; the partial unique index makes SCD2 corruption structurally impossible. |
| **Classify-once dead-letter routing (CASE + JSONB)** | Every rejected row keeps its reason and full original content; nothing vanishes. |
| **Audit table with the reconciliation identity** (read = loaded + rejected) | Run history, performance baselines, and silent-loss detection in one table. |
| **Monitoring as queries** | "No success in 26h" and "rejects > 1%" — the two alerts every warehouse needs, expressible in SQL today, pluggable into any tool tomorrow. |
| **Idempotency via transaction-wrapped delete-by-scope / upsert** | Re-runs and double-deliveries are absorbed, and a mid-load crash rolls back cleanly. |
| **Post-load quality report on cross-row invariants** | Catches what row-level constraints can't: orphans, duplicates, arithmetic drift. |
| **Testing every control by breaking it** | Constraints, checks, and alerts are only real once you've seen them fire. |

---

## 6. Reflection

### What you learned
The four defenses of a production warehouse: schema constraints, dead-letter validation, run auditing with reconciliation, and idempotent transactional loads — plus post-load quality reporting and monitoring expressed as SQL.

### Why it matters
Everything before this lab made the warehouse *work*; this lab makes it *trustworthy unattended*. This is precisely the material the SK-02 capstone grades hardest, and the vocabulary (dead-letter, audit trail, idempotent load, quality gate) recurs through SK-03's dbt tests and Airflow, SK-04's Glue Data Quality, and SK-07.

### Interview questions (with model answers)

1. **"How do you prevent bad data from reaching a fact table?"**
   Layered: validating load routes rule-breaking rows to a dead-letter table with reasons; database constraints (CHECK, FK, partial unique) as the un-bypassable net; post-load quality checks for cross-row invariants; and an audit reconciliation proving read = loaded + rejected.

2. **"What is a dead-letter table and what goes in it?"**
   A quarantine table for rows failing validation: timestamp, source, rule violated, and the full original row (JSONB). Reviewed by humans; recurring patterns become upstream fixes. Rejection without retention is deletion.

3. **"How do you make a SQL batch load idempotent?"**
   Delete-by-scope-then-insert inside a transaction, or `INSERT ... ON CONFLICT`. Scope by the keys present in the batch, not the calendar. Prove it: run twice, compare count + sum.

4. **"Why wrap the delete-then-insert in a transaction?"**
   If the insert fails, rollback restores the deleted rows. Without it, the failure mode of your idempotency mechanism is data loss.

5. **"How would you monitor a nightly warehouse load with no monitoring tools?"**
   An audit table written by the ETL itself, plus two queries: freshness (no success within SLA window) and reject rate above threshold. Any scheduler can poll them and alert; the *logic* is the monitoring, the tool is plumbing.

6. **"Foreign keys in a warehouse — yes or no?"**
   In PostgreSQL at moderate volume: yes, cheap insurance. In columnar MPP warehouses (Redshift/Snowflake) they're unenforced, so validation must move into load-time checks — which is why the check-based pattern matters everywhere.

### Common interview traps
- "Constraints will catch it" *alone* — an unhandled constraint violation aborts the whole nightly load over one bad row. Pair with triage.
- Idempotency by `TRUNCATE` + full reload — valid for small tables, but say why it stops scaling (hours of reload, downtime window) and name the incremental alternative.
- Quality checks that only run once, manually. They run after *every* load, and their results are recorded.

### Key takeaways
1. Constraints stop what code misses; checks triage what constraints would abort.
2. Dead letters keep the evidence; the audit table keeps the reconciliation.
3. Idempotent = transaction + delete-by-batch-scope (or upsert), proven by the run-twice ritual.
4. Monitoring starts as two SQL queries, not a product purchase.

**Next:** the [SK-02 capstone project](../Project/01_Business_Scenario.md) — a full warehouse engagement graded on exactly these defenses.
