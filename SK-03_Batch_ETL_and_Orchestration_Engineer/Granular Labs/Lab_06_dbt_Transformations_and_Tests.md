# Lab 06 — dbt Transformations & Tests

> **Module:** SK-03 Batch ETL & Orchestration Engineer
> **Estimated time:** 4–6 hours
> **Prerequisites:** Lab 05 — your Airflow DAG orchestrates the Spark pipeline writing partitioned Parquet to MinIO, with a Postgres warehouse database available from the docker-compose stack.

---

## 1. Environment Setup

dbt runs against a SQL database. We'll use the Postgres container already in your compose stack (Airflow's metadata Postgres works for learning, but a dedicated warehouse DB is cleaner — your Lab 00 stack includes one; if not, we add it below).

### 1.1 Add a warehouse database (skip if you already have one)

In `docker-compose.yml`, alongside your existing services:

```yaml
  warehouse:
    image: postgres:16
    environment:
      POSTGRES_USER: dwh
      POSTGRES_PASSWORD: dwh_password   # local learning only — never a real password
      POSTGRES_DB: analytics
    ports:
      - "5433:5432"        # 5433 on your laptop to avoid clashing with Airflow's Postgres
    volumes:
      - warehouse_data:/var/lib/postgresql/data
```

Add `warehouse_data:` under `volumes:`, then:

```powershell
docker compose up -d warehouse
docker compose ps
```

**Expected output:** the `warehouse` container `running (healthy)` on port 5433.

### 1.2 Install dbt locally

dbt is a Python tool — install it in a venv on your laptop (it connects *into* the container's Postgres):

```powershell
python -m venv .venv-dbt
.\.venv-dbt\Scripts\Activate.ps1
pip install dbt-postgres
dbt --version
```

**Expected output:** dbt Core and the postgres adapter versions (any current 1.x).

### 1.3 Seed the warehouse with pipeline output

dbt transforms data already *in* the warehouse (that's the ELT pattern). Load your Lab 05 pipeline's output into a `raw` schema. Quick loader script `load_raw.py` (adapt to your Parquet output):

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://dwh:dwh_password@localhost:5433/analytics")
df = pd.read_parquet("output/sales/")          # your Lab 05 partitioned output
df.to_sql("sales", engine, schema="raw", if_exists="replace", index=False)
print(f"Loaded {len(df):,} rows into raw.sales")
```

**Verify:**

```powershell
docker exec -it <warehouse-container> psql -U dwh -d analytics -c "SELECT count(*) FROM raw.sales;"
```

### 1.4 Common installation problems

| Symptom | Fix |
|---|---|
| `dbt: command not recognized` | Activate `.venv-dbt` first |
| `connection refused` on 5433 | `docker compose ps` — is `warehouse` up? Port mapping correct? |
| `relation "raw.sales" does not exist` | The loader script didn't run / wrong schema — create schema first: `CREATE SCHEMA IF NOT EXISTS raw;` |
| Compile error mentioning `profiles.yml` | Step 1 below creates it — dbt needs it before anything works |

---

## 2. Business Context

**The problem dbt solves:** In Lab 05 (and SK-02) your warehouse SQL lived in `.sql` files run in a fixed order by hand or by Airflow. That works — until you have 50 transformation scripts with implicit dependencies, no tests, no documentation, and a new teammate asking "which script builds `revenue_daily`, and what feeds it?"

**dbt (data build tool)** industrializes warehouse SQL: each transformation is a version-controlled `SELECT` (a **model**), dependencies are declared with `ref()` so dbt derives the execution order automatically, tests live next to the models, and documentation generates itself with a clickable lineage graph.

**Where it's used:** dbt is the de-facto standard transformation layer of the modern data stack — thousands of companies run their analytics transformations this way, usually orchestrated by Airflow (exactly what you'll wire up in Step 6). "dbt experience" appears in a large share of data engineering job postings.

**Who consumes the output?** Analysts and BI tools read the **marts** dbt builds; engineers read its lineage and docs; the on-call reads its test results. **If it fails silently**, dashboards show wrong revenue — dbt's built-in testing exists precisely so that failure is loud instead.

---

## 3. Concept Explanation

### 3.1 Models, ref(), and source()

A **model** is one `SELECT` statement in a `.sql` file; dbt materializes it as a view or table named after the file.

- `{{ source('raw', 'sales') }}` — declares "this reads external raw data" (documented + testable).
- `{{ ref('stg_sales') }}` — declares "this reads another model". From these declarations dbt builds the dependency graph (DAG — same concept as Airflow's, at SQL granularity) and always runs things in the right order. Delete-your-run-order-README energy.

### 3.2 Staging vs marts (the standard project shape)

| Layer | Naming | Purpose | Materialization |
|---|---|---|---|
| **Staging** | `stg_*` | 1:1 with each source table: rename, cast, light cleaning — nothing else | `view` (cheap, always fresh) |
| **Marts** | `fct_*`, `dim_*` | Business entities: facts and dimensions, joins and aggregations | `table` (fast for BI) |

This mirrors your medallion thinking: staging ≈ silver, marts ≈ gold. One rule keeps projects sane: **marts never read sources directly; only staging reads sources.**

### 3.3 Materializations

`view` (stored query, always current, compute on read) vs `table` (built on each `dbt run`, fast to read) vs `incremental` (only processes new rows — the dbt version of your high-water-mark loads). Start with view/table; reach for incremental when build times hurt.

### 3.4 Tests

- **Generic (schema) tests** — declared in YAML per column: `not_null`, `unique`, `accepted_values`, `relationships` (FK check). Four keywords cover ~90% of real quality checks.
- **Singular tests** — a `.sql` file returning *rows that violate a rule*; zero rows = pass. Your SK-02 quality-report queries translate directly.

`dbt test` runs them all and exits non-zero on failure — which is exactly the hook Airflow needs to gate the pipeline.

---

## 4. Step-by-Step Implementation

### Step 1 — Initialize the project and connection

**What:**

```powershell
dbt init freshflow_analytics
```

When prompted choose `postgres`. Then edit the generated `~\.dbt\profiles.yml`:

```yaml
freshflow_analytics:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      port: 5433
      user: dwh
      password: dwh_password
      dbname: analytics
      schema: analytics
      threads: 4
```

**Why `profiles.yml` lives outside the project:** it holds credentials, and the project folder goes into version control. Same separation as `.env` in SK-01 — dbt bakes the practice into its design. (In production, values come from environment variables: `password: "{{ env_var('DBT_PASSWORD') }}"`.)

**Verify:**

```powershell
cd freshflow_analytics
dbt debug
```

**Expected output:** `All checks passed!` — connection, profile, and project all valid. If `connection failed`: wrong port (5433) or the container is down.

### Step 2 — Declare your source

**What:** Create `models/staging/_sources.yml`:

```yaml
version: 2

sources:
  - name: raw
    description: "Output of the Spark pipeline (Lab 05), loaded from MinIO parquet"
    schema: raw
    tables:
      - name: sales
        description: "One row per order line, all stores"
        loaded_at_field: processed_at
        freshness:
          warn_after: {count: 26, period: hour}
```

**Why:** sources make the external boundary explicit, documented, and monitorable — the `freshness` block gives you `dbt source freshness`, the "did last night's data actually arrive?" check, for free. That's the SK-02 26-hour alert, productized.

### Step 3 — Build staging models

**What:** Create `models/staging/stg_sales.sql`:

```sql
with source as (
    select * from {{ source('raw', 'sales') }}
),

renamed as (
    select
        order_id,
        store_id,
        product_id,
        cast(quantity as integer)          as quantity,
        cast(unit_price as numeric(10,2))  as unit_price,
        cast(quantity as integer) * cast(unit_price as numeric(10,2))
                                           as line_amount,
        cast(order_ts as timestamp)        as ordered_at,
        date(cast(order_ts as timestamp))  as order_date
    from source
)

select * from renamed
```

And `models/staging/_staging.yml` configuring the layer:

```yaml
version: 2

models:
  - name: stg_sales
    description: "Sales order lines: typed, renamed, one row per order line"
    columns:
      - name: order_id
        tests: [not_null]
      - name: quantity
        tests: [not_null]
```

**Why the CTE style** (`source` → `renamed` → final select): it's dbt's community convention — every model reads top-to-bottom as a pipeline, and each CTE is inspectable by selecting from it during development.

**Run and verify:**

```powershell
dbt run --select stg_sales
```

**Expected output:** `Completed successfully`, `1 of 1 OK created sql view model analytics.stg_sales`. Check it: `SELECT * FROM analytics.stg_sales LIMIT 5;` via psql.

**Common mistake:** putting business logic (joins, aggregations) in staging. Staging is *shape hygiene only* — the moment you join two sources you're writing a mart.

### Step 4 — Build the marts

**What:** `models/marts/fct_daily_store_sales.sql`:

```sql
{{ config(materialized='table') }}

with sales as (
    select * from {{ ref('stg_sales') }}
)

select
    order_date,
    store_id,
    count(distinct order_id)  as orders,
    sum(quantity)             as units_sold,
    sum(line_amount)          as revenue
from sales
group by 1, 2
```

**Why `ref()` and not `analytics.stg_sales`:** `ref()` is what builds the dependency graph, keeps environments portable (dev/prod schemas differ), and makes the lineage documentation truthful. Hardcoding schema-qualified names silently breaks all three — the most common beginner error in dbt.

**Run the whole project:**

```powershell
dbt run
```

**Expected output:** staging view + mart table built *in dependency order* — you never specified the order; `ref()` did.

### Step 5 — Add tests, then break them

**What:** `models/marts/_marts.yml`:

```yaml
version: 2

models:
  - name: fct_daily_store_sales
    description: "Daily revenue per store — feeds the ops dashboard"
    columns:
      - name: revenue
        tests:
          - not_null
      - name: store_id
        tests:
          - not_null
```

Plus one **singular test**, `tests/assert_no_negative_revenue.sql`:

```sql
-- Rows returned = failures. Negative daily revenue means bad source data
-- or a broken transformation upstream.
select *
from {{ ref('fct_daily_store_sales') }}
where revenue < 0
```

**Run:**

```powershell
dbt test
```

**Expected output:** all tests `PASS`. Now the ritual you know from SK-01/SK-02 — prove the alarm works:

```sql
-- via psql: poison one raw row inside a transaction you can undo
INSERT INTO raw.sales (order_id, store_id, product_id, quantity, unit_price, order_ts)
VALUES ('TEST-BAD', 'S01', 'P01', -999, 10.0, now());
```

```powershell
dbt run; dbt test
```

**Expected:** `assert_no_negative_revenue` FAILS, and dbt's exit code is non-zero (`echo $LASTEXITCODE` → `1`). Delete the poison row, rerun, green again. That non-zero exit is what Airflow will act on.

### Step 6 — Generate docs and lineage

**What:**

```powershell
dbt docs generate
dbt docs serve
```

**Expected output:** a browser opens on a documentation site: every model, column, description, test — and the **lineage graph** button (bottom right) showing `raw.sales → stg_sales → fct_daily_store_sales`.

**Why this matters more than it looks:** this is the "which script builds revenue and what feeds it?" answer, generated from code that *cannot drift* from reality (it's derived from `ref()`/`source()`, not hand-drawn). Client handovers in the capstone should include a screenshot of this graph.

### Step 7 — Orchestrate dbt from Airflow

**What:** add two tasks to your Lab 05 DAG, downstream of the Spark load:

```python
from airflow.providers.standard.operators.bash import BashOperator

dbt_run = BashOperator(
    task_id="dbt_run",
    bash_command="cd /opt/airflow/dbt/freshflow_analytics && dbt run --profiles-dir /opt/airflow/dbt",
)

dbt_test = BashOperator(
    task_id="dbt_test",
    bash_command="cd /opt/airflow/dbt/freshflow_analytics && dbt test --profiles-dir /opt/airflow/dbt",
)

load_to_warehouse >> dbt_run >> dbt_test
```

Mount the dbt project into the Airflow containers (in docker-compose, add to the airflow service volumes):

```yaml
      - ./freshflow_analytics:/opt/airflow/dbt/freshflow_analytics
      - ./dbt-profiles:/opt/airflow/dbt   # a profiles.yml using host 'warehouse', port 5432
```

**Why the second profiles file:** inside the Docker network, the warehouse is reachable as `warehouse:5432`, not `localhost:5433` — a classic container-networking gotcha worth experiencing deliberately. Also install `dbt-postgres` into the Airflow image (add to your Dockerfile/requirements from Lab 04).

**Verify:** trigger the DAG. The run should flow Spark → load → `dbt_run` → `dbt_test`, all green. Then re-poison a row and trigger again: `dbt_test` fails, the DAG run is marked failed, and — because tests sit *before* anything downstream would publish — the gate held. This is your SK-01 quality gate, SK-02 quality report, and Airflow orchestration converging into one pattern.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Transformations as version-controlled SELECTs with declared dependencies** | Execution order derived, not maintained by hand; lineage always truthful. |
| **Staging/marts layering; only staging touches sources** | Change isolation: a renamed source column touches one stg model, not fifty. |
| **Credentials outside the project (`profiles.yml`, `env_var()`)** | Same secrets discipline as SK-01, enforced by tool design. |
| **Tests adjacent to models; non-zero exit on failure** | Quality gating becomes a one-line orchestration hook. |
| **Source freshness checks** | "Did the data even arrive?" — monitored declaratively. |
| **Generated docs + lineage** | Handover documentation that cannot go stale. |
| **dbt run/test as DAG tasks after load** | The full modern ELT shape: orchestrator → load → transform → test-gate. |

---

## 6. Reflection

### What you learned
dbt's core loop — model, ref, run, test, document — layered as staging and marts, connected to your Dockerized warehouse, quality-gated, and orchestrated from Airflow.

### Why it matters
Airflow + Spark + dbt is arguably the most common modern batch stack; you have now built all three and wired them together. The capstone expects exactly this shape.

### Interview questions (with model answers)

1. **"What is dbt and what problem does it solve?"**
   A transformation framework for SQL in the warehouse (the T of ELT): models as version-controlled SELECTs, dependencies via `ref()` giving automatic ordering and lineage, tests in YAML/SQL, generated docs. It replaces folders of hand-ordered scripts.

2. **"Explain staging vs mart models."**
   Staging: one per source table, rename/cast/light-clean only, usually views. Marts: business entities (facts/dims) built from staging via `ref()`, usually tables. Sources are only read by staging — isolation that contains upstream change.

3. **"How does dbt know what order to run models in?"**
   From `ref()`/`source()` calls it compiles a DAG and topologically sorts it. No run-order file exists to go stale.

4. **"How do you test in dbt, and how does that integrate with orchestration?"**
   Generic YAML tests (not_null, unique, accepted_values, relationships) plus singular SQL tests returning violating rows. `dbt test` exits non-zero on failure, so an Airflow task after `dbt run` becomes the pipeline's quality gate.

5. **"View vs table vs incremental materialization?"**
   View: always fresh, compute on read — default for staging. Table: rebuilt each run, fast reads — marts. Incremental: processes only new rows for large facts — the high-water-mark pattern; needs a unique key and careful late-data handling.

6. **"Where do dbt credentials live?"**
   `profiles.yml` outside the repo, with `env_var()` for secrets in CI/production. Never in the project folder.

### Common interview traps
- Calling dbt an orchestrator or an ETL tool — it *only* transforms inside the warehouse; Airflow schedules it, something else loads.
- Hardcoding table names instead of `ref()` — interviewers probe *why* ref matters (graph, environments, lineage).
- "dbt tests replace data quality tools" — they overlap but freshness/volume/anomaly monitoring can need more; know the boundary.

### Key takeaways
1. `ref()` is the whole trick: declared dependencies → derived order, lineage, docs.
2. Staging shields marts from source churn.
3. `dbt test`'s exit code turns quality into an orchestration gate.
4. You now hold the full modern batch stack: Docker infra, Spark processing, Airflow orchestration, dbt transformation + testing.

**Next:** the [SK-03 capstone project](../Project/01_Business_Scenario.md).
