# Lab 06 — Warehouse, Shortcuts and SQL Views

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Give MedVista's T-SQL analysts the Gold star schema without copying it: OneLake shortcuts into a serving lakehouse in `medvista-analytics`, a Fabric Warehouse that queries it with three-part names, and three analytical views.
> **Time:** 2–3 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab_05](Lab_05_Pipeline_Orchestration_Alerts_and_Monitoring.md) — Gold holds 3,119 encounters and 755 supply rows; the `medvista-analytics` workspace from Lab_00 exists
> **Guided lab coverage:** Part 1 Step 1.1 (analytics items), Part 5 Steps 5.1 (shortcuts) and 5.2 (views)
> **Cost:** $0 on the trial. Warehouse queries consume capacity only while they run.

---

## Learning objectives

By the end of this lab you can:

- Explain the difference between a lakehouse's SQL analytics endpoint and a Fabric Warehouse.
- Create OneLake table shortcuts and explain why they aren't copies.
- Explain why a Warehouse can't query a lakehouse in another workspace directly, and how shortcuts solve it.
- Write cross-database T-SQL with three-part names, and create schemas and views in a Warehouse.
- Use Fabric's `MEDIAN()` aggregate and the `LAG` window function in a view.

---

## 1. Environment Setup

Verify Lab_05:

1. In a notebook: `spark.table("medvista_gold.dbo.fact_encounters").count()` → `3119`.
2. Workspace `medvista-analytics` exists and is on the trial (its **+ New item** list includes **Warehouse**).

---

## 2. Business Context

The CMO's quality team are SQL people. They know `JOIN` and `GROUP BY`; they don't know PySpark and shouldn't need access to raw PHI in Bronze. They need:

- daily hospital performance with the CMS readmission status,
- year-over-year supply cost by category,
- readmissions by hospital and diagnosis.

They work in `medvista-analytics`. The data engineering workspace stays engineers-only.

---

## 3. Concept Explanation

### 3.1 SQL analytics endpoint vs Warehouse

| | Lakehouse SQL analytics endpoint | Fabric Warehouse |
|---|---|---|
| Created | Automatically with each lakehouse | Explicitly, as its own item |
| Tables | Read-only views of the lakehouse's Delta tables (written by Spark) | Read/write T-SQL tables (`INSERT`, `UPDATE`, `MERGE`) |
| Views, functions, procedures | Yes | Yes |
| Storage | Delta in OneLake | Delta in OneLake |

Here the data is written by Spark, so the Warehouse holds **views only**. It's the analysts' front door: one place for their objects, permissions and query history.

### 3.2 Cross-database queries — same workspace only

A Warehouse can query other warehouses and SQL analytics endpoints **in the same workspace** using three-part names: `database.schema.table`. Gold lives in `medvista-data-engineering`; the Warehouse lives in `medvista-analytics`. So you bring Gold *into* the analytics workspace — without copying it.

### 3.3 OneLake shortcuts

A **shortcut** is a pointer in one lakehouse to data stored somewhere else: another lakehouse (any workspace), ADLS Gen2, Amazon S3, and more. A table shortcut to `medvista_gold/Tables/dbo/fact_encounters` looks and queries like a local table, but no data is copied: when tomorrow's pipeline updates Gold, the shortcut shows the new rows immediately. Shortcuts are read-only through the shortcut.

```
medvista-data-engineering                         medvista-analytics
  medvista_gold.Lakehouse                           medvista_gold_serving.Lakehouse
    Tables/dbo/fact_encounters  ◄── shortcut ───      Tables/dbo/fact_encounters
    Tables/dbo/dim_hospital     ◄── shortcut ───      Tables/dbo/dim_hospital  ...
                                                    medvista_warehouse.Warehouse
                                                      analytics.vw_* ──► medvista_gold_serving.dbo.*
```

**Security note:** OneLake checks permissions on the shortcut **target** as well, so before go-live test the views with a real analyst account that has a role only in `medvista-analytics`, and adjust per your organisation's OneLake security model.

### 3.4 T-SQL details in Fabric Warehouse

- `CREATE SCHEMA` and `CREATE VIEW` must each be the only statement in their batch. In the Fabric query editor, highlight one statement and run it; `IF SCHEMA_ID(...) IS NULL EXEC('CREATE SCHEMA ...')` makes the schema step re-runnable.
- `CREATE OR ALTER VIEW` makes view scripts re-runnable.
- `MEDIAN(expr)` is an aggregate in Fabric Warehouse and SQL analytics endpoints. (In SQL Server, `PERCENTILE_CONT` is analytic-only and needs `OVER`, so it can't sit next to `GROUP BY`.)
- Delta `BOOLEAN` columns appear as `bit`: test with `= 1`.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the serving lakehouse and table shortcuts

1. `medvista-analytics` → **+ New item** → **Lakehouse** → `medvista_gold_serving` (schemas ticked).
2. In its explorer: **Tables** → **dbo** → **…** → **New table shortcut** → **Microsoft OneLake**.
3. Pick **medvista_gold** (in `medvista-data-engineering`) → **Next**.
4. Expand **Tables → dbo** and tick `fact_encounters`, `fact_supply_costs`, `dim_hospital`, `dim_date` → **Next** → **Create**.

**Expected output:** four tables under `dbo`, each with a small link icon.

✅ **Checkpoint:** right-click `fact_encounters` → **Properties** shows a shortcut target pointing at `medvista_gold`.

### Step 2 — Query the shortcuts through the SQL analytics endpoint

Switch `medvista_gold_serving` to its **SQL analytics endpoint** (top-right selector) → **New SQL query**:

```sql
SELECT 'fact_encounters' AS t, COUNT(*) AS n FROM dbo.fact_encounters UNION ALL
SELECT 'fact_supply_costs', COUNT(*) FROM dbo.fact_supply_costs UNION ALL
SELECT 'dim_hospital', COUNT(*) FROM dbo.dim_hospital UNION ALL
SELECT 'dim_date', COUNT(*) FROM dbo.dim_date;
```

**Expected output:** 3,119 · 755 · 12 · 1,461.

If a table is missing or a count is 0, wait a minute and refresh: the SQL analytics endpoint syncs Delta metadata in the background.

✅ **Checkpoint:** the four counts match Gold.

### Step 3 — Create the Warehouse and test a cross-database query

1. `medvista-analytics` → **+ New item** → **Warehouse** → `medvista_warehouse`.
2. In the Warehouse Explorer, select **+ Warehouses** and add `medvista_gold_serving`.
3. **New SQL query**:

   ```sql
   SELECT TOP 5 h.hospital_name, COUNT(*) AS encounters
   FROM medvista_gold_serving.dbo.fact_encounters f
   JOIN medvista_gold_serving.dbo.dim_hospital h ON f.hospital_key = h.hospital_key
   GROUP BY h.hospital_name
   ORDER BY encounters DESC;
   ```

**Expected output:** MedVista Downtown 549, MedVista West Campus 497, MedVista Northside 374, …

Now try going straight to Gold from here:

```sql
SELECT COUNT(*) FROM medvista_gold.dbo.fact_encounters;
```

**Expected output:** an error — `medvista_gold` isn't in this workspace. That's the problem the shortcuts solve.

✅ **Checkpoint:** three-part names work for `medvista_gold_serving` and fail for `medvista_gold`.

### Step 4 — Create the schema and the three views

Run each statement on its own (highlight → **Run**). This is the guided lab's Step 5.2 script:

```sql
-- ════════════════════════════════════════════════════════════════════
-- STEP 0: Schema for analyst-facing objects
-- ════════════════════════════════════════════════════════════════════

IF SCHEMA_ID('analytics') IS NULL
    EXEC('CREATE SCHEMA analytics');


-- ════════════════════════════════════════════════════════════════════
-- VIEW 1: Hospital Performance Summary (daily grain)
-- Used by: CMO monthly dashboard, CMS compliance reports
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_hospital_performance AS
SELECT
    h.hospital_name,
    h.region,
    h.hospital_type,
    h.bed_count,
    d.full_date        AS report_date,
    d.month_name,
    d.year_number,
    d.quarter_name,

    COUNT(DISTINCT f.encounter_id)                          AS encounter_count,
    COUNT(DISTINCT f.patient_id)                            AS unique_patients,
    AVG(f.length_of_stay_hours)                             AS avg_los_hours,
    MEDIAN(f.length_of_stay_hours)                          AS median_los_hours,

    -- CMS Readmission metric
    SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END)  AS readmission_count,
    CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
        / NULLIF(COUNT(DISTINCT f.encounter_id), 0)         AS readmission_rate,

    -- CMS status (target < 15.5% for most conditions)
    CASE
        WHEN CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
             / NULLIF(COUNT(DISTINCT f.encounter_id), 0) > 0.155
        THEN 'ABOVE TARGET' ELSE 'WITHIN TARGET'
    END                                                     AS cms_readmission_status,

    SUM(f.total_charges)                                    AS total_charges,
    AVG(f.total_charges)                                    AS avg_charge_per_encounter

FROM medvista_gold_serving.dbo.fact_encounters   f
JOIN medvista_gold_serving.dbo.dim_hospital      h ON f.hospital_key = h.hospital_key
JOIN medvista_gold_serving.dbo.dim_date          d ON f.admission_date_key = d.date_key
GROUP BY
    h.hospital_name, h.region, h.hospital_type, h.bed_count,
    d.full_date, d.month_name, d.year_number, d.quarter_name;


-- ════════════════════════════════════════════════════════════════════
-- VIEW 2: Supply Cost Year-over-Year Analysis
-- Used by: CFO supply cost dashboard, contract negotiation prep
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_supply_cost_yoy AS
WITH monthly AS (
    SELECT
        h.hospital_name, h.region, s.category_group,
        d.year_number, d.month_number, d.month_name,
        SUM(s.total_cost)     AS monthly_cost,
        SUM(s.total_quantity) AS monthly_quantity
    FROM medvista_gold_serving.dbo.fact_supply_costs s
    JOIN medvista_gold_serving.dbo.dim_hospital      h ON s.hospital_key = h.hospital_key
    JOIN medvista_gold_serving.dbo.dim_date          d ON s.date_key = d.date_key
    GROUP BY h.hospital_name, h.region, s.category_group,
             d.year_number, d.month_number, d.month_name
),
with_lag AS (
    SELECT
        hospital_name, region, category_group,
        year_number, month_number, month_name,
        monthly_cost, monthly_quantity,
        LAG(monthly_cost) OVER (
            PARTITION BY hospital_name, category_group, month_number
            ORDER BY year_number
        ) AS prior_year_cost
    FROM monthly
)
SELECT
    hospital_name, region, category_group,
    year_number, month_number, month_name,
    monthly_cost, monthly_quantity, prior_year_cost,
    CASE
        WHEN prior_year_cost IS NOT NULL AND prior_year_cost > 0
        THEN ROUND((monthly_cost - prior_year_cost) / prior_year_cost * 100, 2)
        ELSE NULL
    END AS yoy_cost_change_pct
FROM with_lag;


-- ════════════════════════════════════════════════════════════════════
-- VIEW 3: Readmission Analysis (monthly, by hospital and diagnosis)
-- Used by: CMS quarterly submission, quality-improvement teams
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_readmission_analysis AS
SELECT
    h.hospital_name,
    h.region,
    d.year_number,
    d.quarter_name,
    d.month_number,
    d.month_name,
    f.encounter_type,
    f.primary_diagnosis,
    COUNT(*)                                                AS encounter_count,
    SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END)  AS readmission_count,
    CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
        / NULLIF(COUNT(*), 0)                               AS readmission_rate,
    AVG(f.length_of_stay_hours)                             AS avg_los_hours
FROM medvista_gold_serving.dbo.fact_encounters   f
JOIN medvista_gold_serving.dbo.dim_hospital      h ON f.hospital_key = h.hospital_key
JOIN medvista_gold_serving.dbo.dim_date          d ON f.admission_date_key = d.date_key
GROUP BY
    h.hospital_name, h.region, d.year_number, d.quarter_name,
    d.month_number, d.month_name, f.encounter_type, f.primary_diagnosis;
```

**Expected output:** each statement completes; in the Explorer, **Schemas → analytics → Views** lists `vw_hospital_performance`, `vw_readmission_analysis` and `vw_supply_cost_yoy`.

### Step 5 — Verify the views

```sql
SELECT COUNT(*) AS daily_rows FROM analytics.vw_hospital_performance;
-- 36 (12 hospitals × 3 days)

SELECT SUM(readmission_count) AS readmissions, SUM(encounter_count) AS encounters
FROM analytics.vw_readmission_analysis;
-- 246 | 3119

SELECT hospital_name, report_date, readmission_rate
FROM analytics.vw_hospital_performance
WHERE cms_readmission_status = 'ABOVE TARGET';
-- 1 row: MedVista Urgent S | 2026-03-19 | 0.1607  (9 readmissions out of 56)

SELECT h.hospital_name, MEDIAN(f.length_of_stay_hours) AS median_los_hours
FROM medvista_gold_serving.dbo.fact_encounters f
JOIN medvista_gold_serving.dbo.dim_hospital h ON f.hospital_key = h.hospital_key
WHERE h.hospital_id IN ('H001', 'H005')
GROUP BY h.hospital_name;
-- MedVista Downtown 3.95 | MedVista West Campus 4.31

SELECT TOP 5 hospital_name, category_group, month_name, monthly_cost, prior_year_cost, yoy_cost_change_pct
FROM analytics.vw_supply_cost_yoy ORDER BY monthly_cost DESC;
-- prior_year_cost and yoy_cost_change_pct are NULL: there's no March 2025 data yet
```

✅ **Checkpoint:** all five results match the comments.

### Step 6 — Prove shortcuts are live, not copies

Run the pipeline once more for `2026-03-19` (Lab_05) — or just note the time of Gold's last change in `DESCRIBE HISTORY`. Re-run `SELECT MAX(etl_loaded_at) FROM medvista_gold_serving.dbo.fact_encounters;` in the Warehouse: it moves forward without you touching the analytics workspace.

### Step 7 — (Optional) Give an analyst access

**Workspace** `medvista-analytics` → **Manage access** → add a colleague (or a test account) as **Viewer**. Ask them to run `SELECT TOP 10 * FROM analytics.vw_hospital_performance`. If it fails on the shortcut, that's the OneLake target-permission check from Section 3.3 — note what extra permission your organisation's model needs.

### Step 8 — (Optional, useful for the capstone) A Direct Lake semantic model

The guided lab stops at T-SQL views, but the capstone asks for a Power BI report in **Direct Lake** mode, and the Gold star schema is ready for it:

1. Open `medvista_gold_serving` → **New semantic model** → name `MedVista Gold` → tick the four tables → **Confirm**. A semantic model created from a lakehouse uses Direct Lake: Power BI reads the Delta files in OneLake directly, without an import refresh.
2. Open the model → **Open data model**. Create relationships: `fact_encounters[hospital_key]` → `dim_hospital[hospital_key]`, `fact_encounters[admission_date_key]` → `dim_date[date_key]`, `fact_supply_costs[hospital_key]` → `dim_hospital[hospital_key]`, `fact_supply_costs[date_key]` → `dim_date[date_key]` (all many-to-one, single direction).
3. Add a measure on `fact_encounters`: `Readmission Rate = DIVIDE(CALCULATE(COUNTROWS(fact_encounters), fact_encounters[is_readmission] = TRUE()), COUNTROWS(fact_encounters))`.
4. **New report** → a bar chart of Readmission Rate by `dim_hospital[hospital_name]`. The two urgent-care sites lead (MedVista Urgent S 13.0%, MedVista Urgent N 10.3%), followed by MedVista West Campus (9.1%); the overall rate is 7.9%.

SK-06 covers semantic modelling and DAX in depth.

---

## 5. Production Engineering Practices

1. **Shortcut, don't copy.** One copy of Gold means one version of the truth and no sync jobs.
2. **Views are a contract.** Analysts depend on view column names; change Gold freely behind them, but version view changes and announce them.
3. **Business-friendly columns in views**: `hospital_name`, `report_date` — not `hospital_key`.
4. **Re-runnable DDL** (`IF SCHEMA_ID ... IS NULL`, `CREATE OR ALTER VIEW`) so deployment scripts can run in any environment.
5. **Workspace per audience.** Analysts get roles in `medvista-analytics` only; access to PHI in Bronze stays with engineers.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Invalid object name 'medvista_gold_serving.dbo.fact_encounters'` | Lakehouse not added to the Warehouse explorer, or typo | **+ Warehouses** → add `medvista_gold_serving`; check spelling |
| Shortcut table shows 0 rows / is missing in SQL | Metadata sync lag | Wait a minute and refresh; re-open the endpoint |
| `CREATE SCHEMA must be the first statement in a query batch` | Ran the whole script at once | Highlight and run statements one at a time (the `EXEC(...)` form avoids this for the schema) |
| `'MEDIAN' is not a recognized built-in function name` | Query running somewhere other than a Fabric Warehouse/SQL endpoint (e.g. SQL Server) | Run it in Fabric; elsewhere use `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x) OVER (PARTITION BY ...)` with `DISTINCT` |
| `New table shortcut` not offered under `dbo` | Lakehouse created without schemas | Use **Tables → New shortcut** instead; tables then appear without a schema folder, still queryable as `dbo.<table>` |
| Analyst gets a permission error on a view | OneLake checks the shortcut target's permissions | Follow your organisation's OneLake security guidance; grant the minimum needed on `medvista_gold` |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"Lakehouse SQL endpoint or Warehouse?"** — Endpoint when Spark writes the data and SQL users only read it. Warehouse when you need T-SQL writes (`INSERT`/`MERGE`, stored-procedure ELT) or a dedicated home for SQL objects and permissions. Both store Delta in OneLake.
2. **"How do you let a team in another workspace query your Gold tables without copying them?"** — OneLake shortcuts in a lakehouse in their workspace, with views on top; manage permissions on both sides.
3. **"Why does the year-over-year view return NULLs?"** — `LAG` over `year_number` needs the same month in the prior year; there's only March 2026. The view is correct; the data history is short.

### Common interview traps

- Assuming a shortcut is a scheduled copy. It's a pointer; reads always see the current target.
- Using `SELECT *` in analyst views: columnar storage only skips columns you don't select, and `*` silently changes the view's shape when Gold changes.

### Key takeaways

- Cross-database T-SQL works within a workspace; shortcuts bring data across workspaces without copying.
- The Warehouse is the analysts' contract: schema `analytics`, re-runnable views, friendly names.
- `MEDIAN()` and `LAG` let views answer the CMO's questions directly.

---

## Next

[Lab_07 — Real-Time Intelligence: Eventstreams and KQL](Lab_07_Real_Time_Intelligence_Eventstreams_and_KQL.md): stream bed events into an Eventhouse and answer "how many beds are free right now?"
