# Lab 07 — Quality Gate, Branching and Warehouse Load

> **Goal:** Add the guided lab's quality gate (`BranchPythonOperator` reading `metrics.json`), the abort-and-alert branch, the warehouse load `TaskGroup` (delete-then-copy), the success alert and a join node with the right trigger rule — then watch a good day load and a bad day get stopped.
> **Time:** 180–240 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 06
> **Guided lab coverage:** Part 2 Step 2.2 (`_quality_gate`: `S3Hook` read, `xcom_push`, branch return values) and Step 2.3 tasks 4–6 (`quality_gate`, `alert_quality_failure`, `load_to_redshift` TaskGroup with `truncate_staging` + `copy_to_staging`, `alert_success`, `trigger_rule`); Reflection Question 1

## Learning objectives

By the end of this lab you can:

- Make an orchestrator *decide* based on data, with `BranchPythonOperator`.
- Pull values from XCom in a later task.
- Group related tasks with a `TaskGroup`, and address a grouped task as `group_id.task_id`.
- Load a warehouse **idempotently** (delete the day, then insert it) and explain the Redshift `COPY` it stands in for.
- Explain trigger rules — `all_success`, `none_failed_min_one_success` — and what "skipped" does to downstream tasks.

## Steps

### Step 1 — Create the `warehouse_default` connection

The guided lab loads Amazon Redshift through `redshift_default`; locally the warehouse is Postgres:

```powershell
docker exec -it airflow-scheduler airflow connections add warehouse_default --conn-type postgres --conn-host warehouse --conn-port 5432 --conn-login dwh_user --conn-password dwh_pass --conn-schema warehouse
```

**Expected output:**

```
Successfully added `conn_id`=warehouse_default : postgres://dwh_user:******@warehouse:5432/warehouse
```

Note the host is the **service name** `warehouse` and port **5432** — Airflow is inside the Docker network. (From Windows you would use `localhost:5433`.)

✅ **Checkpoint:** Admin → Connections shows `warehouse_default`.

### Step 2 — Write the loader (local stand-in for Redshift `COPY`)

`staging.daily_orders` already exists — Lab 00's `warehouse_init.sql` created it with exactly the columns of the Spark job's clean output. Create `C:\DataEngineering\sk03\jobs\load_to_warehouse.py`:

```python
"""Load one day of processed orders from the data lake into staging.daily_orders.

Local stand-in for Redshift's COPY. Idempotent on its own: deletes the day's
rows and inserts them again inside ONE transaction.

Usage: spark-submit /opt/airflow/jobs/load_to_warehouse.py --process_date 2026-07-01
"""
import argparse
import os
import sys

import psycopg2
from psycopg2.extras import execute_values
from pyspark.sql import functions as F

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from etl_job import PipelineConfig, create_spark_session  # reuse the same config/session

COLUMNS = [
    "order_id", "customer_id", "platform", "order_date", "product_id", "product_name",
    "category", "quantity", "unit_price", "discount_pct", "shipping_cost", "tax_amount",
    "payment_method", "shipping_country", "shipping_state", "shipping_city", "is_returned",
    "gross_revenue", "discount_amount", "net_revenue", "total_amount",
    "order_year", "order_month", "order_day", "order_hour", "day_of_week", "is_weekend",
    "order_item_count", "order_total_revenue", "is_high_value",
    "etl_processed_at", "etl_process_date",
]


def main(process_date: str) -> None:
    year, month, day = (int(p) for p in process_date.split("-"))
    spark = create_spark_session(f"load_to_warehouse_{process_date}")
    try:
        df = (
            spark.read.parquet(f"{PipelineConfig.S3_PROCESSED}/orders/")
            # Partition pruning: Spark only opens the folder for this date
            .filter((F.col("order_year") == year)
                    & (F.col("order_month") == month)
                    & (F.col("order_day") == day))
            .select(*COLUMNS)
        )
        rows = [tuple(r) for r in df.collect()]   # OK for ~2,000 rows; see lab notes
    finally:
        spark.stop()

    conn = psycopg2.connect(
        host=os.environ["WAREHOUSE_HOST"], port=os.environ["WAREHOUSE_PORT"],
        dbname=os.environ["WAREHOUSE_DB"], user=os.environ["WAREHOUSE_USER"],
        password=os.environ["WAREHOUSE_PASSWORD"],
    )
    try:
        with conn, conn.cursor() as cur:          # one transaction: all or nothing
            cur.execute("DELETE FROM staging.daily_orders WHERE etl_process_date = %s",
                        (process_date,))
            deleted = cur.rowcount
            execute_values(
                cur,
                f"INSERT INTO staging.daily_orders ({', '.join(COLUMNS)}) VALUES %s",
                rows,
            )
    finally:
        conn.close()
    print(f"[LOAD] dt={process_date} deleted={deleted} inserted={len(rows)}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--process_date", required=True, help="YYYY-MM-DD")
    main(parser.parse_args().process_date)
```

- **Reuses `PipelineConfig` and `create_spark_session`** from `etl_job.py` — one source of truth for paths and S3 settings.
- **Partition pruning:** reading `processed/orders/` and filtering on `order_year/order_month/order_day` makes Spark open only that day's folder.
- **One transaction:** `with conn:` commits the `DELETE` and `INSERT` together or not at all. Running it twice for a date gives the same rows — idempotent on its own.
- **`collect()` is acceptable here only** because a day is ~2,000 rows. At real volume you bulk-load: Redshift `COPY` straight from S3 (the guided lab), Postgres `COPY`, or Spark's JDBC writer.

> **On AWS (guided lab):** `copy_to_staging` is an `S3ToRedshiftOperator` that issues `COPY staging.daily_orders FROM 's3://urbangear-data-lake/processed/orders/order_year=.../' FORMAT AS PARQUET`. Redshift reads the Parquet files directly from S3 — no data passes through Airflow.

**Expected output:** none yet.

✅ **Checkpoint:** `jobs\load_to_warehouse.py` saved.

### Step 3 — Run the loader twice by hand

```powershell
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/load_to_warehouse.py --process_date 2026-07-01
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/load_to_warehouse.py --process_date 2026-07-01
docker exec -it warehouse psql -U dwh_user -d warehouse -c "SELECT etl_process_date, count(*) AS rows FROM staging.daily_orders GROUP BY 1 ORDER BY 1;"
```

**Expected output** (Spark log lines omitted):

```
[LOAD] dt=2026-07-01 deleted=0 inserted=1909
[LOAD] dt=2026-07-01 deleted=1909 inserted=1909
 etl_process_date | rows
------------------+------
 2026-07-01       | 1909
(1 row)
```

The second run deleted the 1,909 rows the first run inserted, then inserted them again: still 1,909. That is the guided lab's "DELETE today's partition first (idempotent — safe to re-run)" in action.

✅ **Checkpoint:** `deleted=1909 inserted=1909` on the second run and 1,909 rows in the table.

### Step 4 — Add the quality-gate and alert callables

Open `dags\urbangear_daily_sales_pipeline.py`. Directly **above** the line `# ── DAG ───...`, add (guided lab Step 2.2):

```python
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
```

- **`hook.read_key(...)`** downloads `metrics/orders/dt=<ds>/metrics.json` — the sidecar `etl_job.py` wrote in Lab 04.
- **A branch callable returns the `task_id` to run next**; every other direct downstream task is **skipped**. Because the load tasks will live in a `TaskGroup` named `load_to_warehouse`, the id is `"load_to_warehouse.truncate_staging"`.
- **`xcom_push(key=...)`** stores `reject_rate`, `clean_count` and `rejected_count`; the alert tasks pull them with `xcom_pull(task_ids="quality_gate", key=...)`.
- The gate passes when `reject_rate <= 5%` and fails when it is **greater than** 5% — the docstring and the code say the same thing.
- **`AirflowFailException`** in `_alert_quality_failure` marks the task failed *without retries*, so an aborted run shows **red** in the UI. That task gets `on_failure_callback=None` so the alert isn't sent twice.

**Expected output:** none yet.

✅ **Checkpoint:** the three functions sit above the `# ── DAG` line.

### Step 5 — Add tasks 4–7 and the new dependencies

Inside the `with DAG(...)` block, **below** the `run_spark_etl` task, add (guided lab Step 2.3, tasks 4–4b):

```python
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
```

Then add the success alert and the join node:

```python
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
```

Finally **replace** the dependency line `wait_for_files >> check_source >> run_spark_etl` with:

```python
    # ── Task dependency graph ────────────────────────────────────
    (
        wait_for_files
        >> check_source
        >> run_spark_etl
        >> quality_gate
        >> [alert_quality_failure, load_to_warehouse]
    )
    load_to_warehouse >> alert_success
    [alert_quality_failure, alert_success] >> pipeline_end
```

(Lab 08 inserts dbt between `load_to_warehouse` and `alert_success`.)

How the **trigger rules** play out:

| Gate result | `alert_quality_failure` | `load_to_warehouse.*` | `alert_success` (default `all_success`) | `pipeline_end` (`none_failed_min_one_success`) | Run |
|---|---|---|---|---|---|
| PASS (≤ 5%) | skipped | success | success | **success** (one branch succeeded, none failed) | success |
| FAIL (> 5%) | runs, alerts, **fails** | skipped | skipped (upstream skipped) | upstream_failed | **failed** |

With the default `all_success`, `pipeline_end` would *never* run on a good day, because one of its parents (`alert_quality_failure`) is always skipped. `none_failed_min_one_success` is the standard rule for a task that joins branches. (This is guided lab Reflection Question 1.)

<details><summary>Your complete DAG file after this step (click to compare)</summary>

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
    load_to_warehouse >> alert_success
    [alert_quality_failure, alert_success] >> pipeline_end
```

</details>

Check it imports:

```powershell
docker exec -it airflow-scheduler airflow dags list-import-errors
```

**Expected output:** `No data found`

✅ **Checkpoint:** no import errors; the Graph view shows the branch, the `load_to_warehouse` group box, and `pipeline_end` joining both paths.

### Step 6 — A good day: the gate passes and the warehouse loads

```powershell
docker exec -it airflow-scheduler airflow dags unpause urbangear_daily_sales_pipeline
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-03
```

Watch the Grid view, then open the logs of `quality_gate`, `copy_to_staging` and `alert_success`.

**Expected output** (in the task logs):

```
[QUALITY GATE] PASSED — reject_rate=2.69%
...
[LOAD] dt=2026-07-03 deleted=0 inserted=1950
...
[ALERT] :white_check_mark: *UrbanGear Daily Pipeline Complete*
*Date:* 2026-07-03
*Clean records:* 1950
```

In the Grid: every task green except `alert_quality_failure`, which is **pink (skipped)**; `pipeline_end` is green.

✅ **Checkpoint:** the run succeeded, `alert_quality_failure` was skipped, 1,950 rows were loaded.

### Step 7 — A bad day: the gate stops the load

2026-07-05 was generated with 12% dirty data (Lab 01).

```powershell
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-05
```

**Expected output** (task logs):

```
[QUALITY GATE] FAILED — reject_rate=11.78% > 5%
...
[ALERT] :x: *UrbanGear Pipeline ABORTED — Quality Gate Failed*
*Date:* 2026-07-05
*Reject Rate:* 11.78%
*Action:* Investigate s3://urbangear-data-lake/rejected/orders/dt=2026-07-05/
...
airflow.exceptions.AirflowFailException: Quality gate failed: reject_rate=11.78%
```

In the Grid: `run_spark_etl` and `quality_gate` green, `alert_quality_failure` **red**, the `load_to_warehouse` tasks and `alert_success` **pink (skipped)**, `pipeline_end` **orange (upstream_failed)**; the run is **failed**.

Confirm nothing from 07-05 reached the warehouse:

```powershell
docker exec -it warehouse psql -U dwh_user -d warehouse -c "SELECT count(*) FROM staging.daily_orders WHERE etl_process_date = '2026-07-05';"
```

**Expected output:**

```
 count
-------
     0
(1 row)
```

The Spark job *did* process 07-05 (its clean rows and rejects are in MinIO for investigation) — the gate's job is to stop bad data from reaching the **warehouse**, where dashboards read it.

✅ **Checkpoint:** a red, failed run for 07-05 and `count 0` in the warehouse.

### Step 8 — Load the remaining good days

Load the other processed days so Lab 08 has data. PowerShell loop:

```powershell
foreach ($d in "2026-07-02", "2026-07-04", "2026-07-06") {
  docker exec airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/load_to_warehouse.py --process_date $d
}
docker exec -it warehouse psql -U dwh_user -d warehouse -c "SELECT etl_process_date, count(*) AS rows FROM staging.daily_orders GROUP BY 1 ORDER BY 1;"
```

**Expected output** (Spark log lines omitted):

```
[LOAD] dt=2026-07-02 deleted=0 inserted=1924
[LOAD] dt=2026-07-04 deleted=0 inserted=1933
[LOAD] dt=2026-07-06 deleted=0 inserted=1948
 etl_process_date | rows
------------------+------
 2026-07-01       | 1909
 2026-07-02       | 1924
 2026-07-03       | 1950
 2026-07-04       | 1933
 2026-07-06       | 1948
(5 rows)
```

Pause the DAG again: `docker exec -it airflow-scheduler airflow dags pause urbangear_daily_sales_pipeline`.

✅ **Checkpoint:** five days, 9,664 rows in `staging.daily_orders`, nothing for 07-05.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `quality_gate` fails: `Could not read quality metrics` | No `metrics.json` for that date | `run_spark_etl` must have succeeded; check `metrics/orders/dt=<date>/` in MinIO |
| Error `'truncate_staging' is not a valid task id` / branch skips everything | Branch returned the id without the group prefix | Return `"load_to_warehouse.truncate_staging"` |
| `pipeline_end` skipped on a good day | Join node left on the default trigger rule | `trigger_rule="none_failed_min_one_success"` |
| `truncate_staging`: `relation "staging.daily_orders" does not exist` | Warehouse volume existed before Lab 00's init SQL | `docker exec -i warehouse psql -U dwh_user -d warehouse < sql\warehouse_init.sql` |
| `copy_to_staging`: `could not translate host name "warehouse"` | Script run outside Docker | Run it through `docker exec airflow-scheduler ...` |
| `copy_to_staging`: `column ... does not exist` | `warehouse_init.sql` or `load_to_warehouse.py` differ from the lab | Copy both exactly; columns must match |
| Alerts appear twice for a quality failure | `on_failure_callback=None` missing on `alert_quality_failure` | Add it |
| Want real Slack messages | — | Add `SLACK_WEBHOOK_URL: https://hooks.slack.com/...` to the `x-airflow-common` environment in `docker-compose.yml`, then `docker compose up -d` |

## Key takeaways

- The Spark job measures quality; the orchestrator **decides** — `BranchPythonOperator` reads `metrics.json` and picks the load path or the abort path.
- Branch callables return task ids; tasks inside a `TaskGroup` are `group_id.task_id`.
- Loads are idempotent: delete the day, then insert/copy it, in one transaction.
- Skipped tasks propagate: anything after a skipped branch is skipped unless its trigger rule says otherwise. Join nodes use `none_failed_min_one_success`.
- A failed quality gate should produce a red run *and* an alert — never a quiet green.

## Next

[Lab 08 — dbt Models, Tests and the Full Pipeline](Lab_08_dbt_Models_Tests_and_Full_Pipeline.md): build `stg_orders` and `fct_daily_sales` as incremental dbt models with tests, wire `run_dbt_transformations` into the DAG, and run the guided lab's verification queries.
