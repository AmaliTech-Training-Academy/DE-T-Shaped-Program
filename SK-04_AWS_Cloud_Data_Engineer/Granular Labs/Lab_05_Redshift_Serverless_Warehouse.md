# Lab 05 — Redshift Serverless Warehouse

> **Module:** SK-04 AWS Cloud Data Engineer
> **Estimated time:** 4–5 hours
> **Cost estimate:** $0–4. Redshift Serverless bills per RPU-second of actual query work; new accounts often get free trial credits that cover this lab. **Teardown at the end is mandatory either way.**
> **Prerequisites:** Lab 04 — the SwiftHaul data lake has curated Parquet in S3, cataloged by Glue, queryable in Athena.

---

## 1. Environment Setup

### 1.1 Verify prerequisites

```powershell
aws sts get-caller-identity
aws s3 ls s3://swifthaul-lake-<your-suffix>-curated/ --recursive --human-readable | Select-Object -First 5
aws glue get-tables --database-name swifthaul_curated --query "TableList[].Name"
```

**Expected output:** your IAM identity, curated Parquet objects, and the Glue tables from Lab 04. If any fail, revisit that lab first — this lab builds directly on those artifacts.

### 1.2 Cost guardrails FIRST (before creating anything)

Redshift Serverless has one number that controls your maximum burn rate: **Base capacity (RPUs)**. We set it to the minimum and add a usage limit.

Check the current minimum in your region (typically 8 RPUs):

```powershell
aws redshift-serverless get-workgroup --workgroup-name default 2>$null   # likely errors — nothing exists yet, that's fine
```

Confirm your Lab 00 budget alert is still active (Billing → Budgets). This lab is exactly why we set it up.

### 1.3 Common setup problems

| Symptom | Fix |
|---|---|
| `AccessDenied` on redshift-serverless calls | Your IAM user needs `AmazonRedshiftFullAccess` (or scoped equivalent) — attach via the console as in Lab 01 |
| Region mismatch (resources "missing") | `aws configure get region` — everything in this module lives in ONE region |
| Curated bucket empty | Re-run the Lab 04 Glue job |

---

## 2. Business Context

**Why a warehouse when Athena already queries the lake?** SwiftHaul's ops team hits Athena a few times a day — per-query pricing and lake-speed latency are fine. But the finance team is about to launch a **dashboard refreshed every few minutes, powering 40 concurrent users at month-end close**. That workload profile — repeated, concurrent, latency-sensitive BI queries over modeled data — is what a **data warehouse** is for.

| | Athena (lake query) | Redshift (warehouse) |
|---|---|---|
| Pricing | per TB scanned | per compute-time (Serverless) or per cluster-hour |
| Best at | ad-hoc exploration, infrequent queries | repeated BI queries, concurrency, joins over modeled schemas |
| Data | wherever it lies in S3 | loaded (COPY) or referenced (Spectrum) |
| Latency | seconds+ | sub-second to seconds on tuned tables |

**Who consumes it:** the finance dashboard (and in this module's project, *your* Power BI dashboard). **If it's slow or wrong:** month-end close stalls and forty people notice simultaneously — warehouse workloads fail loudly and publicly, which is why load discipline (idempotent COPY, validation) matters as much as speed.

**Industry reality:** the lake+warehouse combination you're building — raw/processed/curated in S3, modeled marts in Redshift — is one of the most common enterprise data architectures on AWS.

---

## 3. Concept Explanation

### 3.1 Redshift Serverless vs provisioned clusters

Classic Redshift is a **provisioned cluster**: nodes running (and billing) 24/7. **Serverless** replaces that with per-second **RPU** (Redshift Processing Unit) billing only while queries run — dramatically cheaper for learning and spiky workloads. Two objects to know:

- **Namespace** — the durable part: databases, users, encryption. (Storage: ~$0.024/GB-month — pennies here.)
- **Workgroup** — the compute part: base RPU capacity, network settings, usage limits.

### 3.2 COPY: bulk loading done right

`COPY` is Redshift's parallel bulk loader — it reads many files from S3 simultaneously across compute slices. The cardinal rule: **never load row-by-row INSERTs into Redshift** (thousands of times slower; a classic real-world incident pattern). COPY from Parquet also carries the schema, avoiding delimiter/quoting drama.

### 3.3 Distribution and sort keys (the two Redshift-specific tuning ideas)

Redshift is **MPP** (massively parallel processing): tables are sliced across compute nodes.

- **DISTKEY** decides *which rows live on which slice*. Joining two tables distributed on the join key = no data shuffling (fast). `DISTSTYLE AUTO` is the sensible default; know *why* you'd override it.
- **SORTKEY** decides *row order within storage blocks*, letting Redshift skip blocks whose min/max don't match your filter — the columnar cousin of partitioning. Date columns are the classic choice for time-filtered BI.

### 3.4 Redshift Spectrum

Spectrum lets Redshift query **external tables that stay in S3** (via the Glue catalog you already built) and join them with local tables. The standard pattern: hot, modeled data loaded locally; long-tail history queried in place. It's also the cheapest way to *get* data into Redshift: `INSERT INTO local SELECT ... FROM spectrum_table`.

### 3.5 Materialized views

A **materialized view** stores a query's *results* and can refresh on demand — precisely what a dashboard should read, so the expensive aggregation runs once per refresh instead of once per user per visit.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the namespace and workgroup (console)

**What:** Console → **Amazon Redshift** → **Serverless dashboard** → create:

- Namespace: `swifthaul-ns`; database `swifthaul_dw`; admin user `dwadmin` with a password you generate and store properly (Lab 00 discipline — password manager, never a file in your repo).
- **Associated IAM role:** create/attach a role with S3 read on your lake buckets + `AWSGlueConsoleFullAccess` (Spectrum needs the catalog) and set it as the namespace's **default role**.
- Workgroup: `swifthaul-wg`; **Base capacity: 8 RPU** (the minimum); publicly accessible: **off**.

**Why the default IAM role matters:** COPY and Spectrum run *as the warehouse*, not as you — the namespace role is how Redshift reaches S3/Glue. Nine out of ten first-COPY failures are this role missing or unattached.

**Add the spend guardrail** — workgroup → **Limits** → add usage limit: RPU-hours, **4 per day**, action **Turn off enhanced… / Alert + Log** (choose *Alert* + a second one that *deactivates*): even a forgotten runaway query cannot exceed ~a few dollars.

**Verify:** workgroup shows `Available`. **Expected time:** ~3–5 minutes to provision.

**Common mistake:** enabling "publicly accessible" to make connecting easier. Don't — the Query Editor v2 in the console needs no public endpoint, and public warehouses are a top real-world breach cause.

### Step 2 — Connect and create the schema

**What:** Console → Redshift → **Query editor v2** → connect to `swifthaul-wg` / `swifthaul_dw`. Run:

```sql
CREATE SCHEMA IF NOT EXISTS marts;

CREATE TABLE marts.fct_shipments (
    shipment_id      VARCHAR(32)   NOT NULL,
    shipped_date     DATE          NOT NULL,
    origin_hub       VARCHAR(8),
    dest_hub         VARCHAR(8),
    carrier_id       VARCHAR(16),
    weight_kg        DECIMAL(10,2),
    revenue_usd      DECIMAL(12,2),
    delivered_hours  DECIMAL(8,2)
)
DISTSTYLE AUTO
SORTKEY (shipped_date);
```

(Adapt columns to your Lab 04 curated schema — reconciling DDL to real data is part of the exercise.)

**Why `SORTKEY (shipped_date)`:** the dashboard filters by date ranges; sorting by date lets Redshift skip most blocks. **Why `DISTSTYLE AUTO`:** Redshift adapts distribution as the table grows; override only when profiling proves a join-shuffle problem.

**Verify:** `SELECT * FROM marts.fct_shipments;` returns zero rows, no error.

### Step 3 — Load with COPY, idempotently

**What:**

```sql
BEGIN;

DELETE FROM marts.fct_shipments;   -- full-refresh scope for this lab; see Why below

COPY marts.fct_shipments
FROM 's3://swifthaul-lake-<suffix>-curated/shipments/'
IAM_ROLE default
FORMAT AS PARQUET;

COMMIT;

-- load audit — same discipline as SK-02
SELECT query_id, table_name, status, row_count
FROM sys_load_history
ORDER BY start_time DESC LIMIT 5;
```

**Why delete-then-COPY in a transaction:** the by-now-familiar idempotency pattern — re-running the load cannot duplicate rows, and a mid-load failure rolls back. At production scale you'd scope the DELETE to the arriving partition's dates (exactly as in SK-02 Lab 06); full-refresh is honest for this table size.

**Expected output:** COPY completes with a row count matching your curated data. Verify the reconciliation:

```sql
SELECT count(*) FROM marts.fct_shipments;
```

against the Athena count of the same curated table — **cross-system reconciliation**, your strongest defense against silent partial loads.

**Troubleshooting:**
| Error | Cause → fix |
|---|---|
| `S3ServiceException: Access Denied` | Namespace default IAM role missing S3 read → attach policy, re-run |
| `Spectrum/COPY: no files found` | Wrong prefix — `aws s3 ls` the exact path |
| Column mismatch | Parquet schema ≠ DDL — compare with Athena's `DESCRIBE`, fix DDL |
| Sits "queued" for minutes | Workgroup paused/limits hit — check workgroup status and usage limits |

### Step 4 — Spectrum: query the lake without loading it

**What:**

```sql
CREATE EXTERNAL SCHEMA IF NOT EXISTS lake
FROM DATA CATALOG DATABASE 'swifthaul_curated'
IAM_ROLE default;

-- Local (loaded) vs external (in-place) — same SQL surface:
SELECT count(*) FROM lake.shipments;         -- scans S3 via Spectrum
SELECT count(*) FROM marts.fct_shipments;    -- local storage

-- The join pattern that makes the architecture sing:
SELECT f.carrier_id, count(*) AS shipments, avg(l.weight_kg) AS avg_weight
FROM marts.fct_shipments f
JOIN lake.shipments l USING (shipment_id)
GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
```

**Why:** you now have both halves of the lakehouse trade-off under your fingers — loaded-and-fast vs in-place-and-cheap — and can articulate when each wins (the finance dashboard reads `marts.*`; the once-a-quarter historical analysis reads `lake.*`).

**Cost note:** Spectrum bills ~$5/TB scanned *in addition* to RPU time — partition pruning and Parquet matter here exactly as they did in Athena.

### Step 5 — The dashboard's materialized view

**What:**

```sql
CREATE MATERIALIZED VIEW marts.mv_daily_carrier_kpis AS
SELECT
    shipped_date,
    carrier_id,
    count(*)                    AS shipments,
    sum(revenue_usd)            AS revenue,
    avg(delivered_hours)        AS avg_delivery_hours,
    sum(weight_kg)              AS total_weight_kg
FROM marts.fct_shipments
GROUP BY 1, 2;

SELECT * FROM marts.mv_daily_carrier_kpis ORDER BY shipped_date DESC LIMIT 10;

-- after each load:
REFRESH MATERIALIZED VIEW marts.mv_daily_carrier_kpis;
```

**Why:** this MV is what the module project's **Power BI dashboard** will query — one pre-aggregated, cheap-to-read table instead of 40 users each re-aggregating the fact table. `REFRESH` after COPY becomes a step in your pipeline (in the project, Terraform + your orchestration make this automatic).

**Verify:** re-run the Step 3 load, `REFRESH` the MV, confirm the numbers are unchanged (idempotency, observed end-to-end).

### Step 6 — TEARDOWN (mandatory)

**What:** Even serverless has storage costs and accident potential. When you finish the lab (and between sessions if you'll pause for days):

```powershell
# Delete workgroup first, then namespace
aws redshift-serverless delete-workgroup --workgroup-name swifthaul-wg
aws redshift-serverless delete-namespace --namespace-name swifthaul-ns --no-final-snapshot
```

*Doing the project soon?* Alternative: keep the namespace, and note that a workgroup with **no running queries bills zero RPU-seconds** — the risk is accidental queries, which your usage limit caps. Decide deliberately and write the decision down (cost decisions are engineering decisions).

**Verify teardown:**

```powershell
aws redshift-serverless list-namespaces
aws redshift-serverless list-workgroups
```

Both empty. Check **Cost Explorer** tomorrow: Redshift line ≤ a few dollars, then zero.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Guardrails before resources** (min RPU, usage limits, budget alert) | Cloud cost discipline is a *pre-provisioning* activity; afterwards is archaeology. |
| **Warehouse loads via transactional delete+COPY** | Idempotent, roll-back-able bulk loading — the SK-02 pattern at cloud scale. |
| **Cross-system reconciliation (Athena count vs Redshift count)** | Catches partial COPYs that in-system checks can't see. |
| **Default IAM role for warehouse→lake access** | Services act with their own identities; least-privilege roles, never embedded keys. |
| **Right engine per workload (Athena ad-hoc vs Redshift BI vs Spectrum hybrid)** | Architecture-level cost/performance judgment — a signature senior-engineer skill. |
| **Materialized views as the BI contract** | Dashboards read pre-aggregations; refresh is a pipeline step. |
| **Deliberate teardown with verification** | Unmanaged cloud resources are the #1 source of surprise bills. |

---

## 6. Reflection

### What you learned
Provisioning cost-guarded Redshift Serverless, loading idempotently with COPY, distribution/sort key reasoning, querying the lake in place with Spectrum, serving BI from a materialized view, and tearing it all down verifiably.

### Why it matters
Lake + warehouse is the dominant AWS analytics architecture; knowing *when to load vs query in place*, and how to keep a warehouse load safe and reconciled, is exactly the judgment interviews for cloud DE roles test.

### Interview questions (with model answers)

1. **"Athena vs Redshift — when each?"**
   Athena: ad-hoc, infrequent, pay-per-scan on the lake. Redshift: repeated, concurrent, latency-sensitive BI on modeled data. Spectrum bridges: hot data local, history in place. Decide by workload profile, not fashion.

2. **"How do you load data into Redshift efficiently?"**
   Parallel `COPY` from S3 (Parquet ideally), never row INSERTs; inside a transaction with delete-by-scope for idempotency; verified against source counts; via the namespace IAM role, not credentials.

3. **"Explain DISTKEY and SORTKEY."**
   DISTKEY: row placement across slices — co-locate join keys to avoid shuffles. SORTKEY: on-disk order enabling block skipping on range filters (dates). Defaults (AUTO) first; override on profiled evidence.

4. **"How do you control Redshift Serverless costs?"**
   Minimum base RPU, workgroup usage limits with alert+deactivate actions, budget alarms, MVs so dashboards don't re-aggregate, and teardown/pause when idle. Name the number: base capacity × RPU-hour price = worst-case burn.

5. **"What's a materialized view and why do dashboards want one?"**
   Stored query results, refreshed on schedule/demand. Forty dashboard users read one precomputed table instead of forty aggregations — cheaper, faster, and load-decoupled.

6. **"A COPY loaded fewer rows than expected and nobody noticed for a week. What controls were missing?"**
   Load auditing (`sys_load_history` review), reconciliation against the source system count, and an alert on the delta — the same read=loaded+rejected identity from warehouse fundamentals.

### Common interview traps
- Recommending Redshift for everything ("it's the warehouse!") — the Athena-vs-Redshift *workload* framing is the expected answer.
- Not knowing Serverless bills RPU-seconds while idle costs ~nothing but storage — cost-model fluency is tested.
- INSERT loops for loading. Instant red flag; the word is COPY.

### Key takeaways
1. Guardrails, then resources. Teardown, then verify.
2. COPY + transaction + reconciliation = trustworthy warehouse loads.
3. Load what's hot, Spectrum what's cold.
4. The dashboard reads a materialized view, not the fact table.

**Next:** [Lab 06 — CloudWatch Monitoring](Lab_06_CloudWatch_Monitoring.md): the pipeline works; now make it observable — logs, metrics, alarms, and the Terraform-provisioned dashboard the project mandates.
