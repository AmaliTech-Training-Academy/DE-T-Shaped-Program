# Lab 08 — dbt Models, Tests and the Full Pipeline

> **Goal:** Build the guided lab's dbt project `urbangear` — source with freshness, incremental `stg_orders` and `fct_daily_sales`, `schema.yml` tests with `dbt_utils` — add `run_dbt_transformations` to the DAG, run the whole pipeline end to end, and run the guided lab's verification queries.
> **Time:** 210–270 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 07
> **Guided lab coverage:** Part 3 Steps 3.1–3.3 (`stg_orders`, `fct_daily_sales`, `schema.yml`, plus the project setup they need: profile, `packages.yml`, `sources.yml`, schema macro), Part 2 Step 2.3 task 5 (`run_dbt_transformations` with `--select tag:daily` and `--vars`), Part 4 (verification queries); Reflection Question 3

## Learning objectives

By the end of this lab you can:

- Set up a dbt project and a profile that reads its credentials from environment variables.
- Declare a **source** with a **freshness** check, and install a package (`dbt_utils`).
- Explain `ref()` / `source()`, staging vs marts, and the `incremental` materialization with a look-back buffer.
- Write generic, package and singular tests, and pass run-time values with `--vars`.
- Select models with **tags** and run dbt from Airflow as the pipeline's last quality gate.

## Steps

### Step 1 — Create the project

All dbt commands run **inside** the scheduler container (dbt is installed there, and it can reach `warehouse:5432`). Open a shell there and stay in it until Step 10:

```powershell
docker exec -it airflow-scheduler bash
```

```bash
cd /opt/airflow/dbt
dbt init urbangear --skip-profile-setup
rm -rf urbangear/models/example
export DBT_PROFILES_DIR=/opt/airflow/dbt/profiles     # where profiles.yml lives
cd urbangear
```

**Expected output** (from `dbt init`, abbreviated):

```
... Running with dbt=1.8.x
... Your new dbt project "urbangear" was created!
```

The project now exists on Windows as `C:\DataEngineering\sk03\dbt\urbangear\` (the `dbt\` folder is mounted into the container) — the guided lab's path `/opt/airflow/dbt/urbangear`. Edit files in VS Code on Windows; run commands in the container shell.

Replace the generated `dbt\urbangear\dbt_project.yml` with:

```yaml
name: 'urbangear'
version: '1.0.0'
profile: 'urbangear'

model-paths: ["models"]
test-paths: ["tests"]
macro-paths: ["macros"]
target-path: "target"
clean-targets: ["target", "dbt_packages"]

vars:
  process_date: null      # set per run: dbt run --vars '{process_date: 2026-07-01}'

models:
  urbangear:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

`vars: process_date: null` declares a variable with a default; `--vars '{process_date: 2026-07-06}'` overrides it per run (Step 7).

✅ **Checkpoint:** `dbt\urbangear\` contains `dbt_project.yml`, `models\` (empty), `macros\`, `tests\`.

### Step 2 — Write the profile and check the connection

Create `C:\DataEngineering\sk03\dbt\profiles\profiles.yml` (the guided lab's `--profiles-dir /opt/airflow/dbt/profiles`):

```yaml
urbangear:
  target: dev
  outputs:
    dev:
      type: postgres
      host: "{{ env_var('WAREHOUSE_HOST') }}"            # 'warehouse' inside Docker
      port: "{{ env_var('WAREHOUSE_PORT') | as_number }}" # 5432 inside Docker
      user: "{{ env_var('WAREHOUSE_USER') }}"
      password: "{{ env_var('WAREHOUSE_PASSWORD') }}"
      dbname: "{{ env_var('WAREHOUSE_DB') }}"
      schema: analytics
      threads: 4
```

Every value comes from the `WAREHOUSE_*` environment variables in `docker-compose.yml` via `env_var()`, so the file holds no secrets and works unchanged in any environment that sets those variables. `schema: analytics` is the default schema for models that don't set their own.

```bash
dbt debug
```

**Expected output** (last lines):

```
  Connection test: [OK connection ok]

All checks passed!
```

✅ **Checkpoint:** `All checks passed!`

### Step 3 — Install `dbt_utils`

The guided lab's tests use `dbt_utils.accepted_range`, which comes from a package. Create `dbt\urbangear\packages.yml`:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
```

```bash
dbt deps
```

**Expected output:**

```
... Installing dbt-labs/dbt_utils
... Installed from version 1.x.x
```

✅ **Checkpoint:** `dbt\urbangear\dbt_packages\dbt_utils\` exists.

### Step 4 — Control schema names with a macro

The guided lab puts `stg_orders` in schema `staging` and `fct_daily_sales` in `analytics` (`config(schema=...)`). By default dbt would *prefix* the profile's schema and create `analytics_staging` and `analytics_analytics`. This small, standard macro makes dbt use the names exactly as written. Create `dbt\urbangear\macros\generate_schema_name.sql`:

```sql
{#
  Use a model's custom schema EXACTLY as written (schema='staging' -> staging).
  Without this macro dbt would prefix the profile schema (analytics_staging).
#}
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

**Expected output:** none yet.

✅ **Checkpoint:** the macro file is saved.

### Step 5 — Declare the source and check its freshness

`source('staging', 'daily_orders')` in the guided lab's `stg_orders` must be declared before it can be used. Create `dbt\urbangear\models\staging\sources.yml`:

```yaml
version: 2

sources:
  - name: staging
    description: "Processed orders loaded from the data lake by the Airflow load step"
    schema: staging
    tables:
      - name: daily_orders
        description: "One row per order line; reloaded per etl_process_date"
        loaded_at_field: etl_processed_at
        freshness:
          warn_after: {count: 26, period: hour}
```

```bash
dbt source freshness
```

**Expected output:** `PASS` if `etl_processed_at` of the newest row is less than 26 hours old, otherwise `WARN`:

```
1 of 1 START freshness of staging.daily_orders ..... [RUN]
1 of 1 PASS freshness of staging.daily_orders ...... [PASS in 0.xxs]
```

If you ran the ETL more than 26 hours ago you'll see `WARN` — exactly the "did last night's data arrive?" alarm this check exists for. Re-run Lab 06's pipeline for any date to refresh it.

✅ **Checkpoint:** freshness shows `PASS` (or a `WARN` you can explain).

### Step 6 — Build the staging and mart models

Create `dbt\urbangear\models\staging\stg_orders.sql` (guided lab Step 3.1):

```sql
-- models/staging/stg_orders.sql
{{
    config(
        materialized = 'incremental',
        unique_key   = ['order_id', 'product_id'],
        schema       = 'staging',
        tags         = ['daily']
    )
}}

WITH source AS (
    SELECT * FROM {{ source('staging', 'daily_orders') }}
    {% if is_incremental() %}
    -- Only process records newer than the latest we already have
    -- The -3 day buffer catches any late-arriving records from recent days
    WHERE etl_processed_at > (
        SELECT COALESCE(MAX(etl_processed_at) - INTERVAL '3 days', '1970-01-01'::TIMESTAMP)
        FROM {{ this }}
    )
    {% endif %}
),

cleaned AS (
    SELECT
        order_id,
        customer_id,
        LOWER(TRIM(platform))                        AS platform,
        order_date,
        product_id,
        TRIM(product_name)                           AS product_name,
        TRIM(category)                               AS category,
        quantity,
        ROUND(unit_price::NUMERIC, 2)                AS unit_price,
        -- NUMERIC (exact decimals) so marts can ROUND(..., 2): Postgres has no
        -- ROUND(double precision, int), and money should not be floating point anyway
        COALESCE(discount_pct,  0)::NUMERIC(5, 2)    AS discount_pct,
        COALESCE(shipping_cost, 0)::NUMERIC(10, 2)   AS shipping_cost,
        COALESCE(tax_amount,    0)::NUMERIC(10, 2)   AS tax_amount,
        UPPER(TRIM(payment_method))                  AS payment_method,
        UPPER(TRIM(shipping_country))                AS shipping_country,
        TRIM(shipping_state)                         AS shipping_state,
        TRIM(shipping_city)                          AS shipping_city,
        COALESCE(is_returned, FALSE)                 AS is_returned,
        etl_processed_at
    FROM source
    WHERE order_id IS NOT NULL
      AND customer_id IS NOT NULL
      AND quantity > 0
)

SELECT * FROM cleaned
```

Create `dbt\urbangear\models\marts\fct_daily_sales.sql` (guided lab Step 3.2):

```sql
-- models/marts/fct_daily_sales.sql
{{
    config(
        materialized = 'incremental',
        unique_key   = ['order_id', 'product_id'],
        schema       = 'analytics',
        tags         = ['daily'],
        post_hook    = "GRANT SELECT ON {{ this }} TO GROUP analytics_readers;"
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    WHERE order_date::date > (
        SELECT COALESCE(MAX(order_date::date) - INTERVAL '3 days', '1970-01-01'::DATE)
        FROM {{ this }}
    )
    {% endif %}
)

SELECT
    order_id,
    customer_id,
    platform,
    order_date,
    product_id,
    product_name,
    category,
    quantity,
    unit_price,
    discount_pct,

    -- Revenue measures
    ROUND(quantity * unit_price,                                2) AS gross_revenue,
    ROUND(quantity * unit_price * discount_pct / 100.0,         2) AS discount_amount,
    ROUND(quantity * unit_price * (1 - discount_pct / 100.0),   2) AS net_revenue,
    shipping_cost,
    tax_amount,
    ROUND(
        quantity * unit_price * (1 - discount_pct / 100.0)
        + shipping_cost + tax_amount,
        2
    )                                                              AS total_amount,

    payment_method,
    shipping_country,
    shipping_state,
    shipping_city,
    is_returned,

    -- Convenience date columns
    DATE_TRUNC('day',   order_date)::DATE AS order_day,
    DATE_TRUNC('week',  order_date)::DATE AS order_week,
    DATE_TRUNC('month', order_date)::DATE AS order_month,
    EXTRACT(DOW  FROM order_date)         AS day_of_week,
    EXTRACT(HOUR FROM order_date)         AS order_hour,
    CASE WHEN EXTRACT(DOW FROM order_date) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend,

    CURRENT_TIMESTAMP AS dbt_updated_at

FROM orders
```

- **Staging vs marts:** `stg_orders` only renames, casts and cleans (one row in, one row out). `fct_daily_sales` adds business measures. Only staging reads the `source`; marts read staging through `ref()` — which is how dbt knows to build `stg_orders` first.
- **Grain:** one row per **order line** — an order can contain several products, so `order_id` alone is *not* unique. The `unique_key` is `['order_id', 'product_id']`.
- **`incremental`:** the first run builds the table from everything; later runs process only rows newer than what the table already holds (`{% if is_incremental() %}` … `{{ this }}`) and merge them by `unique_key` (delete-then-insert on Postgres).
- **The 3-day buffer** (`MAX(etl_processed_at) - INTERVAL '3 days'`) re-processes recent days on every run, so rows that arrive late are still picked up; `unique_key` makes the overlap update rows instead of duplicating them. (Guided lab Reflection Question 3.)
- **`NUMERIC` casts in staging:** money columns become exact decimals. Postgres has no `ROUND(double precision, 2)`, so without them the mart's `ROUND(...)` fails.
- **`post_hook`** grants read access to the `analytics_readers` group created by `warehouse_init.sql`.
- **`tags = ['daily']`** lets Airflow run just these models: `--select tag:daily`.

```bash
dbt run --select tag:daily
```

**Expected output** (timings differ):

```
1 of 2 START sql incremental model staging.stg_orders .................. [RUN]
1 of 2 OK created sql incremental model staging.stg_orders ............. [SELECT 9664 in 0.xxs]
2 of 2 START sql incremental model analytics.fct_daily_sales ........... [RUN]
2 of 2 OK created sql incremental model analytics.fct_daily_sales ...... [SELECT 9664 in 0.xxs]

Completed successfully

Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

9,664 = all clean rows loaded in Lab 07 (five days). Run the same command **again**:

**Expected output:** both models `OK` again (now as incremental merges). Check the row count from a **second** PowerShell window (the Airflow container has no `psql` client; the `warehouse` container does):

```powershell
docker exec -it warehouse psql -U dwh_user -d warehouse -c "SELECT count(*) FROM analytics.fct_daily_sales;"
```

```
 count
-------
  9664
(1 row)
```

✅ **Checkpoint:** 9,664 rows after the first *and* the second run.

### Step 7 — Add the tests and use `--vars`

Create `dbt\urbangear\models\marts\schema.yml` (guided lab Step 3.3):

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: fct_daily_sales
    description: >
      Daily sales fact table — one row per order line item.
      Grain: one product purchased in one order (order_id + product_id).
      Populated by the UrbanGear daily Airflow pipeline.
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns: ['order_id', 'product_id']

    columns:
      - name: order_id
        description: Order identifier (one order can have several line items)
        tests:
          - not_null

      - name: customer_id
        description: Foreign key to the customer dimension
        tests:
          - not_null

      - name: platform
        description: Sales channel (website, mobile_app, marketplace)
        tests:
          - not_null
          - accepted_values:
              values: ['website', 'mobile_app', 'marketplace']

      - name: quantity
        description: Number of units ordered (must be positive)
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 1
              max_value: 10000

      - name: net_revenue
        description: Revenue after discounts (must be >= 0)
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0

      - name: order_date
        description: Timestamp of the order placement
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: "'2020-01-01'"
              max_value: "CURRENT_DATE + INTERVAL '1 day'"
```

Create the singular test `dbt\urbangear\tests\assert_process_date_loaded.sql`:

```sql
-- Singular test: returns a row (= FAIL) if the run's process_date has no rows
-- in the fact table. process_date comes from --vars '{process_date: 2026-07-01}'.
-- Without --vars it checks nothing (returns zero rows = PASS).
{% if var('process_date') %}
SELECT '{{ var("process_date") }}' AS missing_process_date
WHERE NOT EXISTS (
    SELECT 1
    FROM {{ ref('fct_daily_sales') }}
    WHERE order_day = '{{ var("process_date") }}'::date
)
{% else %}
SELECT NULL AS missing_process_date WHERE FALSE
{% endif %}
```

- **Generic tests** (`not_null`, `accepted_values`) are one line of YAML each.
- **Package tests:** `dbt_utils.accepted_range` (min/max) and `dbt_utils.unique_combination_of_columns` (uniqueness of the order-line grain).
- **Singular test:** a `SELECT` that returns the *failing* rows; zero rows = pass. It uses `var('process_date')`, so it checks the date the pipeline just processed.

```bash
dbt test --select tag:daily --vars '{process_date: 2026-07-06}'
```

**Expected output** (last line):

```
Done. PASS=12 WARN=0 ERROR=0 SKIP=0 TOTAL=12
```

Now ask about a date that was **not** loaded (the bad day):

```bash
dbt test --select tag:daily --vars '{process_date: 2026-07-05}'
echo "exit code: $?"
```

**Expected output:**

```
... FAIL 1 assert_process_date_loaded ...
...
Done. PASS=11 WARN=0 ERROR=1 SKIP=0 TOTAL=12
exit code: 1
```

The non-zero exit code is what makes the Airflow task fail.

✅ **Checkpoint:** 12 passes for 07-06; 1 failure and exit code 1 for 07-05.

### Step 8 — Generate the documentation and lineage graph

```bash
dbt docs generate
exit
```

`dbt docs serve` would start a web server *inside* the container that Windows can't reach, so serve the generated site from Windows with Python (any Python 3.12 install from SK-01):

```powershell
python -m http.server 8085 --directory C:\DataEngineering\sk03\dbt\urbangear\target
```

Open http://localhost:8085.

**Expected output:** the dbt docs site, listing `stg_orders` and `fct_daily_sales` with their columns and tests; the lineage graph (bottom-right button) shows `staging.daily_orders → stg_orders → fct_daily_sales`. Press Ctrl+C in PowerShell to stop the server.

✅ **Checkpoint:** you saw the three-node lineage graph.

### Step 9 — Add `run_dbt_transformations` to the DAG

In `dags\urbangear_daily_sales_pipeline.py`, add this task **above** `# ── 6. Success alert` (guided lab Step 2.3, task 5):

```python
    # ── 5. dbt transformations + tests ───────────────────────────
    run_dbt = BashOperator(
        task_id      = "run_dbt_transformations",
        bash_command = (
            "cd /opt/airflow/dbt/urbangear && "
            "dbt run  --select tag:daily --vars '{process_date: {{ ds }}}' "
            "         --profiles-dir /opt/airflow/dbt/profiles && "
            "dbt test --select tag:daily --vars '{process_date: {{ ds }}}' "
            "         --profiles-dir /opt/airflow/dbt/profiles"
        ),
        execution_timeout = timedelta(hours=1),
    )
```

and change the line `load_to_warehouse >> alert_success` to

```python
    load_to_warehouse >> run_dbt >> alert_success
```

- `cd ... && dbt run ... && dbt test ...` — if the run fails, tests don't run; if a test fails, dbt exits 1, the task fails, and `_failure_alert` fires.
- `--vars '{process_date: {{ ds }}}'` — Airflow renders `{{ ds }}`, so dbt (and the singular test) know which day this run processed.
- `--select tag:daily` — only the daily models and their tests.

<details><summary>Your complete DAG file (identical to the guided lab's local DAG)</summary>

```python
"""
UrbanGear Daily Sales Pipeline — Airflow DAG (local stack)
==========================================================
Orchestrates: Wait → Check → Spark ETL → Quality Gate
             → [Load warehouse | Alert failure] → dbt → Success alert

Design principles:
- Idempotent: safe to re-run for any date
- catchup=False: no accidental historical backfill
- max_active_runs=1: prevents overlapping daily runs
- Every task has retries and execution_timeout
"""

import json
import os
from datetime import datetime, timedelta

import requests
from airflow import DAG
from airflow.exceptions import AirflowFailException
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import BranchPythonOperator, PythonOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.utils.task_group import TaskGroup

AWS_CONN_ID       = "aws_default"         # points at MinIO locally (Lab 06)
WAREHOUSE_CONN_ID = "warehouse_default"   # local Postgres warehouse (Lab 07)
S3_BUCKET         = "urbangear-data-lake"
MAX_REJECT_RATE   = 0.05


# ── Alerting ─────────────────────────────────────────────────────
def _send_alert(text: str) -> None:
    """Post to Slack if a webhook URL is configured; always log the alert."""
    webhook = os.environ.get("SLACK_WEBHOOK_URL")
    if webhook:
        requests.post(webhook, json={"text": text}, timeout=10)
    print(f"[ALERT] {text}")


def _failure_alert(context):
    """Callback: fires when any task fails (after its last retry)."""
    ti = context["task_instance"]
    _send_alert(
        ":red_circle: *UrbanGear Pipeline FAILED*\n"
        f"*Task:* `{ti.task_id}`\n"
        f"*Date:* {context['ds']}\n"
        f"*Log:* {ti.log_url}"
    )


DEFAULT_ARGS = {
    "owner":               "data-engineering",
    "depends_on_past":     False,
    "email_on_failure":    False,    # needs SMTP configured; we alert via _failure_alert
    "email_on_retry":      False,
    "retries":             3,
    "retry_delay":         timedelta(minutes=10),
    "execution_timeout":   timedelta(hours=2),
    "on_failure_callback": _failure_alert,
}


# ── Python callables ─────────────────────────────────────────────
def _check_source_data(**context):
    """Count the day's source files and share the number via XCom."""
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook

    process_date = context["ds"]
    hook   = S3Hook(aws_conn_id=AWS_CONN_ID)
    prefix = f"raw/orders/dt={process_date}/"
    keys   = hook.list_keys(bucket_name=S3_BUCKET, prefix=prefix)

    if not keys:
        raise FileNotFoundError(
            f"No source files at s3://{S3_BUCKET}/{prefix}. "
            "Source system export may not have completed."
        )

    context["ti"].xcom_push(key="source_file_count", value=len(keys))
    print(f"[CHECK] {len(keys)} source files found: {keys}")
    return len(keys)


def _quality_gate(**context):
    """
    Read the quality metrics written by the Spark job and choose the next task:
    - 'load_to_warehouse.truncate_staging'  if reject_rate <= 5%
    - 'alert_quality_failure'               if reject_rate >  5%
    """
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook

    process_date = context["ds"]
    hook        = S3Hook(aws_conn_id=AWS_CONN_ID)
    metrics_key = f"metrics/orders/dt={process_date}/metrics.json"

    try:
        metrics = json.loads(hook.read_key(metrics_key, bucket_name=S3_BUCKET))
    except Exception as e:
        raise RuntimeError(f"Could not read quality metrics: {e}")

    reject_rate = metrics.get("reject_rate", 1.0)
    ti = context["ti"]
    ti.xcom_push(key="reject_rate",    value=reject_rate)
    ti.xcom_push(key="clean_count",    value=metrics.get("clean_count", 0))
    ti.xcom_push(key="rejected_count", value=metrics.get("rejected_count", 0))

    if reject_rate > MAX_REJECT_RATE:
        print(f"[QUALITY GATE] FAILED — reject_rate={reject_rate:.2%} > {MAX_REJECT_RATE:.0%}")
        return "alert_quality_failure"

    print(f"[QUALITY GATE] PASSED — reject_rate={reject_rate:.2%}")
    return "load_to_warehouse.truncate_staging"


def _alert_quality_failure(**context):
    reject_rate = context["ti"].xcom_pull(task_ids="quality_gate", key="reject_rate")
    _send_alert(
        ":x: *UrbanGear Pipeline ABORTED — Quality Gate Failed*\n"
        f"*Date:* {context['ds']}\n"
        f"*Reject Rate:* {reject_rate:.2%}\n"
        f"*Action:* Investigate s3://{S3_BUCKET}/rejected/orders/dt={context['ds']}/"
    )
    # Fail on purpose so the run shows red in the UI. AirflowFailException = no retries.
    raise AirflowFailException(f"Quality gate failed: reject_rate={reject_rate:.2%}")


def _alert_success(**context):
    clean_count = context["ti"].xcom_pull(task_ids="quality_gate", key="clean_count")
    _send_alert(
        ":white_check_mark: *UrbanGear Daily Pipeline Complete*\n"
        f"*Date:* {context['ds']}\n"
        f"*Clean records:* {clean_count}"
    )


# ── DAG ──────────────────────────────────────────────────────────
with DAG(
    dag_id          = "urbangear_daily_sales_pipeline",
    default_args    = DEFAULT_ARGS,
    description     = "Daily sales: raw JSON → PySpark → warehouse → dbt → alert",
    schedule        = "0 4 * * *",          # 4:00 AM UTC daily
    start_date      = datetime(2026, 7, 1),
    catchup         = False,                # Never backfill silently
    max_active_runs = 1,                    # Prevent day overlap
    tags            = ["production", "sales", "daily"],
) as dag:

    # ── 1. Sensor: wait up to 1 hour for the day's files ────────
    # mode=reschedule frees the worker slot while waiting.
    wait_for_files = S3KeySensor(
        task_id       = "wait_for_source_files",
        bucket_key    = "raw/orders/dt={{ ds }}/part-*.json",
        wildcard_match = True,                # bucket_key contains a * pattern
        bucket_name   = S3_BUCKET,
        aws_conn_id   = AWS_CONN_ID,
        poke_interval = 300,    # check every 5 minutes
        timeout       = 3600,   # fail if not found within 1 hour
        mode          = "reschedule",
        retries       = 0,      # a sensor timeout is real news — alert, don't retry
    )

    # ── 2. Check source data (count files → XCom) ────────────────
    check_source = PythonOperator(
        task_id         = "check_source_data",
        python_callable = _check_source_data,
    )

    # ── 3. Spark ETL (local mode inside the Airflow container) ───
    run_spark_etl = BashOperator(
        task_id      = "run_spark_etl",
        bash_command = (
            "spark-submit --master 'local[2]' --driver-memory 2g "
            "/opt/airflow/jobs/etl_job.py --process_date {{ ds }}"
        ),
        execution_timeout = timedelta(hours=3),
    )

    # ── 4. Quality gate (branch) ─────────────────────────────────
    quality_gate = BranchPythonOperator(
        task_id         = "quality_gate",
        python_callable = _quality_gate,
    )

    # ── 4a. Abort path: alert on quality failure ─────────────────
    alert_quality_failure = PythonOperator(
        task_id             = "alert_quality_failure",
        python_callable     = _alert_quality_failure,
        on_failure_callback = None,   # it alerts itself; don't alert twice
    )

    # ── 4b. Load path: warehouse ─────────────────────────────────
    with TaskGroup(group_id="load_to_warehouse") as load_to_warehouse:

        # DELETE this date's rows first (idempotent — safe to re-run)
        truncate_staging = SQLExecuteQueryOperator(
            task_id = "truncate_staging",
            conn_id = WAREHOUSE_CONN_ID,
            sql     = "DELETE FROM staging.daily_orders WHERE etl_process_date = '{{ ds }}'::date;",
        )

        # Copy the processed Parquet for this date into staging
        copy_to_staging = BashOperator(
            task_id      = "copy_to_staging",
            bash_command = (
                "spark-submit --master 'local[2]' "
                "/opt/airflow/jobs/load_to_warehouse.py --process_date {{ ds }}"
            ),
        )

        truncate_staging >> copy_to_staging

    # ── 5. dbt transformations + tests ───────────────────────────
    run_dbt = BashOperator(
        task_id      = "run_dbt_transformations",
        bash_command = (
            "cd /opt/airflow/dbt/urbangear && "
            "dbt run  --select tag:daily --vars '{process_date: {{ ds }}}' "
            "         --profiles-dir /opt/airflow/dbt/profiles && "
            "dbt test --select tag:daily --vars '{process_date: {{ ds }}}' "
            "         --profiles-dir /opt/airflow/dbt/profiles"
        ),
        execution_timeout = timedelta(hours=1),
    )

    # ── 6. Success alert ─────────────────────────────────────────
    # Default trigger_rule (all_success): only runs when dbt succeeded.
    # If the gate chose the abort branch, load → dbt → alert_success are all SKIPPED.
    alert_success = PythonOperator(
        task_id         = "alert_success",
        python_callable = _alert_success,
    )

    # ── 7. Join: one end node for both branches ──────────────────
    # none_failed_min_one_success: runs if nothing upstream failed and at least
    # one branch succeeded (the other branch is SKIPPED, which "all_success" would block).
    pipeline_end = EmptyOperator(
        task_id      = "pipeline_end",
        trigger_rule = "none_failed_min_one_success",
    )

    # ── Task dependency graph ────────────────────────────────────
    (
        wait_for_files
        >> check_source
        >> run_spark_etl
        >> quality_gate
        >> [alert_quality_failure, load_to_warehouse]
    )
    load_to_warehouse >> run_dbt >> alert_success
    [alert_quality_failure, alert_success] >> pipeline_end
```

</details>

```powershell
docker exec -it airflow-scheduler airflow dags list-import-errors
```

**Expected output:** `No data found`

✅ **Checkpoint:** the Graph view shows `load_to_warehouse → run_dbt_transformations → alert_success`.

### Step 10 — Run the full pipeline end to end

```powershell
docker exec -it airflow-scheduler airflow dags unpause urbangear_daily_sales_pipeline
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-06
```

**Expected output:** in the Grid view, every task green except `alert_quality_failure` (skipped). The `run_dbt_transformations` log ends with

```
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
...
Done. PASS=12 WARN=0 ERROR=0 SKIP=0 TOTAL=12
```

and `alert_success` logs `*Clean records:* 1948`. Pause the DAG afterwards (`airflow dags pause urbangear_daily_sales_pipeline`).

✅ **Checkpoint:** one fully green run, from sensor to `pipeline_end`.

### Step 11 — Run the guided lab's verification queries

These are guided lab Part 4, with `CURRENT_DATE - 1` replaced by the date you just ran (on a live daily pipeline you'd use yesterday). From PowerShell:

```powershell
docker exec -it warehouse psql -U dwh_user -d warehouse
```

```sql
-- 1. Record count for the run
SELECT order_day, COUNT(*) AS line_item_count, COUNT(DISTINCT order_id) AS order_count,
       COUNT(DISTINCT customer_id) AS unique_customers, ROUND(SUM(net_revenue), 2) AS total_net_revenue
FROM analytics.fct_daily_sales WHERE order_day = DATE '2026-07-06' GROUP BY order_day;

-- 2. Platform distribution
SELECT platform, COUNT(*) AS order_count,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM analytics.fct_daily_sales WHERE order_day = DATE '2026-07-06'
GROUP BY platform ORDER BY order_count DESC;

-- 3. No future-dated orders, 4. no negative revenue (expected: 0 and 0)
SELECT COUNT(*) AS future_order_count FROM analytics.fct_daily_sales WHERE order_date > CURRENT_TIMESTAMP;
SELECT COUNT(*) AS negative_revenue_count FROM analytics.fct_daily_sales WHERE net_revenue < 0;
\q
```

**Expected output** (`total_net_revenue` computed by simulation — treat the last cents as approximate):

```
 order_day  | line_item_count | order_count | unique_customers | total_net_revenue
------------+-----------------+-------------+------------------+-------------------
 2026-07-06 |            1948 |         987 |              368 |         405635.52
(1 row)

  platform   | order_count | pct
-------------+-------------+------
 mobile_app  |         656 | 33.7
 website     |         653 | 33.5
 marketplace |         639 | 32.8
(3 rows)

 future_order_count
--------------------
                  0
(1 row)

 negative_revenue_count
------------------------
                      0
(1 row)
```

✅ **Checkpoint:** 1,948 line items in 987 orders, no future dates, no negative revenue. **You have now built every part of the guided lab.**

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `dbt: command not found` | Shell on Windows, not in the container | `docker exec -it airflow-scheduler bash` first |
| `Could not find profile named 'urbangear'` | `DBT_PROFILES_DIR` not set, or file in the wrong folder | `export DBT_PROFILES_DIR=/opt/airflow/dbt/profiles`; file must be `dbt\profiles\profiles.yml` |
| `Env var required but not provided: 'WAREHOUSE_HOST'` | dbt run somewhere without the compose env vars | Run dbt inside `airflow-scheduler` |
| `'dbt_utils' is undefined` | `dbt deps` not run | `dbt deps` in `/opt/airflow/dbt/urbangear` |
| Models land in `analytics_staging` | Schema macro missing | Add `macros/generate_schema_name.sql` (Step 4) |
| `function round(double precision, integer) does not exist` | Money columns left as floats | Keep the `::NUMERIC(...)` casts in `stg_orders` |
| `role "analytics_readers" does not exist` | `warehouse_init.sql` never ran | `docker exec -i warehouse psql -U dwh_user -d warehouse < sql\warehouse_init.sql` |
| `unique` test fails on `order_id` | Treating an order as one row | The grain is order line: test `order_id` + `product_id` together |
| Row count grows on every `dbt run` | `unique_key` missing | `unique_key = ['order_id', 'product_id']` on both incremental models |
| Logic changed but old rows don't change | Incremental models only touch recent rows | `dbt run --select tag:daily --full-refresh` rebuilds from scratch |

## Key takeaways

- dbt project = `dbt_project.yml` + profile (credentials via `env_var`) + packages + models + tests.
- Sources are declared (with freshness); marts read staging via `ref()`, so dbt derives the build order.
- Incremental models + `unique_key` + a look-back buffer = fast daily runs that still catch late data; `--full-refresh` when logic changes.
- Tests are part of the pipeline: a failing `dbt test` fails the Airflow task and fires the alert.
- `--select tag:daily` and `--vars '{process_date: {{ ds }}}'` connect dbt to the orchestrator's run date.

## Next

You're ready for the [SK-03 End-to-End Guided Lab](../Labs/SK-03_End_to_End_Guided_Lab.md). Every step in it uses something you have built: work through it on this same stack, answer its reflection questions, and read its "On AWS" blocks as a preview of SK-04.
