# Lab 06 — Sensors, Hooks and the Spark Task

> **Goal:** Start the guided lab's `urbangear_daily_sales_pipeline` DAG: wait for the day's files with an `S3KeySensor`, count them with an `S3Hook` (sharing the count via XCom), run `etl_job.py` for `{{ ds }}`, alert on failure — then backfill four days and prove re-runs are safe.
> **Time:** 180–240 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 05
> **Guided lab coverage:** Part 2 Step 2.1 (`AWS_CONN_ID`, `S3_BUCKET`, `_slack_failure_alert` → `_failure_alert`, `DEFAULT_ARGS`), Step 2.2 (`_check_source_data`: `S3Hook.list_keys`, `xcom_push`) and Step 2.3 tasks 1–3 (`wait_for_source_files`, `check_source_data`, `run_spark_etl`); the local equivalent of `GlueJobOperator`

## Learning objectives

By the end of this lab you can:

- Store credentials in an **Airflow Connection** (`aws_default`) instead of in code.
- Use an **S3 hook** to list keys, and a **sensor** to wait for files — and explain `poke` vs `reschedule` mode.
- Push values to **XCom** explicitly with `ti.xcom_push(key=...)`.
- Run a PySpark job from Airflow with `BashOperator` + `spark-submit` (and say what `GlueJobOperator` does instead on AWS).
- Test single tasks with `airflow tasks test`, **backfill** a date range, and prove the pipeline is **idempotent**.

## Steps

### Step 1 — Create the `aws_default` connection (pointing at MinIO)

A **Connection** stores a host, login, password and extras in Airflow's metadata database, encrypted with the Fernet key from Lab 00. DAG code only names the connection id. We use the id `aws_default` — the one the guided lab uses — so S3 sensors and hooks written for AWS work unchanged against MinIO.

The URI form below avoids JSON quoting problems in PowerShell (`%3A%2F%2F` is the URL-encoding of `://`):

```powershell
docker exec -it airflow-scheduler airflow connections add aws_default --conn-uri "aws://minioadmin:minioadmin123@/?endpoint_url=http%3A%2F%2Fminio%3A9000"
docker exec -it airflow-scheduler airflow connections get aws_default -o yaml
```

**Expected output** (abbreviated):

```
Successfully added `conn_id`=aws_default : aws://minioadmin:******@/?endpoint_url=http%3A%2F%2Fminio%3A9000
...
  conn_id: aws_default
  conn_type: aws
  ...
  extra_dejson:
    endpoint_url: http://minio:9000
  ...
  login: minioadmin
```

If it says the connection already exists, run `airflow connections delete aws_default` (same `docker exec` prefix) and add it again.

✅ **Checkpoint:** Admin → Connections in the UI lists `aws_default` with `endpoint_url` in **Extra**.

### Step 2 — Use the S3 hook by hand

A **hook** is Airflow's client for an external system, configured from a Connection. `S3Hook` wraps boto3.

```powershell
docker exec -it airflow-scheduler python
```

```python
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
hook = S3Hook(aws_conn_id="aws_default")
print(hook.list_keys(bucket_name="urbangear-data-lake", prefix="raw/orders/dt=2026-07-01/"))
print(hook.list_keys(bucket_name="urbangear-data-lake", prefix="raw/orders/dt=2026-07-09/"))
print(hook.check_for_wildcard_key("raw/orders/dt=2026-07-02/part-*.json", bucket_name="urbangear-data-lake"))
exit()
```

**Expected output** (possibly after a warning or two):

```
['raw/orders/dt=2026-07-01/part-00000.json']
[]
True
```

`check_for_wildcard_key` is exactly what the sensor will call every few minutes.

✅ **Checkpoint:** one key for 07-01, an empty list for 07-09, `True` for the wildcard.

### Step 3 — Write the pipeline DAG (part 1: wait → check → Spark)

Create `C:\DataEngineering\sk03\dags\urbangear_daily_sales_pipeline.py`. This is the guided lab's DAG with the AWS-only pieces replaced by their local equivalents; Labs 07–08 add the remaining tasks. (A few imports — `BranchPythonOperator`, `EmptyOperator`, `TaskGroup`, `SQLExecuteQueryOperator`, `AirflowFailException` — are used from Lab 07 on; keeping them now saves editing later.)

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

    # ── Task dependency graph ────────────────────────────────────
    wait_for_files >> check_source >> run_spark_etl
```

Read it section by section:

- **`_send_alert` / `_failure_alert`** — the guided lab's `_slack_failure_alert`, made optional: if the environment variable `SLACK_WEBHOOK_URL` is set it posts to Slack, and it always prints an `[ALERT]` line into the task log. `on_failure_callback` in `DEFAULT_ARGS` attaches it to every task.
- **`DEFAULT_ARGS`** — the guided lab's values: 3 retries 10 minutes apart, 2-hour `execution_timeout`. `email_on_failure` is `False` because the local stack has no mail server.
- **`wait_for_source_files` (`S3KeySensor`)** — checks for `raw/orders/dt={{ ds }}/part-*.json` every 5 minutes (`poke_interval=300`) for up to 1 hour (`timeout=3600`). `wildcard_match=True` is **required** because the key contains `*` — without it the sensor looks for a file literally named `part-*.json` and waits forever. `mode="reschedule"` releases the worker slot between checks (`poke` mode would hold it the whole hour). `retries=0`: a timeout is real news — alert now, don't wait another hour.
- **`check_source_data`** — runs *after* the sensor (the files are there, now count them). `xcom_push(key="source_file_count", ...)` stores the count under a named key; the return value is stored too, as `return_value`.
- **`run_spark_etl`** — runs Lab 04's job: `spark-submit --master 'local[2]' ... --process_date {{ ds }}`. The task is a pure function of `{{ ds }}`, so any run date can be (re)processed.

> **On AWS (guided lab):** the same task is a `GlueJobOperator(job_name="urbangear_daily_sales_etl", script_args={"--process_date": "{{ ds }}"}, ...)`, which asks the AWS Glue service to run `etl_job.py` on a managed Spark cluster and waits for it. Locally, `BashOperator` runs the same file inside the Airflow container. Same task id, same job, same argument.

**Expected output:**

```powershell
docker exec -it airflow-scheduler airflow dags list-import-errors
```

```
No data found
```

✅ **Checkpoint:** no import errors; `urbangear_daily_sales_pipeline` appears in the UI.

### Step 4 — Test each task on its own

`airflow tasks test <dag> <task> <date>` runs one task for one logical date, ignores dependencies, and records nothing in the run history — ideal while developing.

```powershell
docker exec -it airflow-scheduler airflow tasks test urbangear_daily_sales_pipeline wait_for_source_files 2026-07-01
docker exec -it airflow-scheduler airflow tasks test urbangear_daily_sales_pipeline check_source_data 2026-07-01
```

**Expected output** (find these lines among the log output):

```
... Poking for key : s3://urbangear-data-lake/raw/orders/dt=2026-07-01/part-*.json
... Success criteria met. Exiting.
...
... [CHECK] 1 source files found: ['raw/orders/dt=2026-07-01/part-00000.json']
... Done. Returned value was: 1
```

Now a date with no files:

```powershell
docker exec -it airflow-scheduler airflow tasks test urbangear_daily_sales_pipeline check_source_data 2026-07-09
```

**Expected output:** the task fails with

```
FileNotFoundError: No source files at s3://urbangear-data-lake/raw/orders/dt=2026-07-09/. Source system export may not have completed.
```

`airflow tasks test` doesn't always run failure callbacks, so you may or may not see the `[ALERT]` block here — in a real run (Step 6, or Lab 07) you always will.

✅ **Checkpoint:** `Returned value was: 1` for 07-01 and `FileNotFoundError` for 07-09.

### Step 5 — Backfill 2026-07-01 to 2026-07-04

A **backfill** creates runs for past logical dates. It is the routine answer to "we deployed today, but we need history" or "we fixed the logic, reprocess last week".

```powershell
docker exec -it airflow-scheduler airflow dags backfill urbangear_daily_sales_pipeline -s 2026-07-01 -e 2026-07-04T04:00:00
```

- Runs of this DAG have logical timestamps at **04:00** (the cron `0 4 * * *`). A bare `-e 2026-07-04` means midnight — *before* the 07-04 run — so it would be skipped. Hence `T04:00:00`.
- `max_active_runs=1` makes the four runs go one after another.
- `catchup=False` doesn't block backfills; it only stops the scheduler from creating past runs *by itself*.

**Expected output:** after a few minutes the command ends with a summary line like

```
... Backfill done for DAG <DAG: urbangear_daily_sales_pipeline>. Exiting.
```

In the UI's Grid view you see four runs (07-01 … 07-04), every task dark green. Open `run_spark_etl` → **Logs** for 07-04 and find `Clean: 1,933 | Rejected: 67 | Rate: 3.35%`.

✅ **Checkpoint:** four green runs; the 07-04 Spark log shows 1,933 clean rows.

### Step 6 — Watch the sensor wait for a late file

Trigger a date whose file doesn't exist yet:

```powershell
docker exec -it airflow-scheduler airflow dags unpause urbangear_daily_sales_pipeline
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-06
```

(`-e` sets the logical date in Airflow 2. The DAG must be unpaused for triggered runs to start. Unpausing also lets the scheduler create the latest *scheduled* run, for yesterday's date; it has no file, so its sensor fails after an hour with an `[ALERT]` — expected, ignore it.) In Grid view, `wait_for_source_files` turns **turquoise** (`up_for_reschedule`): it checked, found nothing, released its slot, and will check again in 5 minutes. Now the file "arrives late":

```powershell
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-06
```

**Expected output:**

```
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-06/part-00000.json (2000 lines, 862,671 bytes)
```

Within 5 minutes the sensor turns green, and `check_source_data` and `run_spark_etl` follow; the Spark log for 07-06 shows `Clean: 1,948 | Rejected: 52 | Rate: 2.60%`.

Pause the DAG again so the scheduler doesn't start daily runs for dates without data:

```powershell
docker exec -it airflow-scheduler airflow dags pause urbangear_daily_sales_pipeline
```

✅ **Checkpoint:** you saw `up_for_reschedule`, then success after the late file arrived.

### Step 7 — Prove idempotency on the record

1. Count 07-02's processed rows:

```powershell
docker exec -it airflow-scheduler python -c "import sys; sys.path.insert(0,'/opt/airflow/jobs'); from etl_job import create_spark_session; s=create_spark_session('idem'); print('rows 07-02:', s.read.parquet('s3a://urbangear-data-lake/processed/orders/').filter('order_day = 2').count()); s.stop()"
```

2. In Grid view, click 07-02's `run_spark_etl` square → **Clear** (it re-runs the task).
3. When it is green again, run the count command again.

**Expected output:** both times

```
rows 07-02: 1924
```

Retries, clears and backfills can never duplicate data, because each run **overwrites a deterministic partition keyed by its logical date**. Say it exactly like that in interviews.

✅ **Checkpoint:** `rows 07-02: 1924` before and after the clear.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Sensor never succeeds although the file exists | `wildcard_match=True` missing, or key pattern wrong | Compare with Step 3: `raw/orders/dt={{ ds }}/part-*.json` + `wildcard_match=True` |
| `Unable to locate credentials` / `InvalidAccessKeyId` | `aws_default` missing or wrong | Re-create it (Step 1); `airflow connections get aws_default` |
| `Could not connect to the endpoint URL: "https://urbangear-data-lake.s3.amazonaws.com/..."` | `endpoint_url` extra missing — boto3 went to real AWS | Re-add the connection with the `?endpoint_url=...` part |
| Backfill creates only 3 runs | End date `2026-07-04` = midnight, before the 04:00 run | Use `-e 2026-07-04T04:00:00` |
| `run_spark_etl` fails with `spark-submit: command not found` | Image not rebuilt after Lab 00 changes | `docker compose up -d --build` |
| Task killed with `AirflowTaskTimeout` | `execution_timeout` too small for your laptop | Normal values: 3 h for Spark; check RAM in `.wslconfig` |
| A failed task retries for 30 minutes | `retries=3`, `retry_delay=10 min` (guided-lab values) | Expected. While developing, Clear and re-run instead of waiting |

## Key takeaways

- Credentials live in Connections (`aws_default`); DAG code only names them.
- Sensor (wait) → hook check (verify, XCom) → compute is the standard start of a batch DAG. Use `mode="reschedule"` and `wildcard_match=True` for `*` patterns.
- `BashOperator` + `spark-submit` locally ≙ `GlueJobOperator` on AWS — same job, same `--process_date {{ ds }}`.
- `airflow tasks test` for one task, `airflow dags backfill` for history (mind the 04:00 logical time), **Clear** to re-run.
- Idempotency is proven by re-running and counting, not by assertion.

## Next

[Lab 07 — Quality Gate, Branching and Warehouse Load](Lab_07_Quality_Gate_Branching_and_Warehouse_Load.md): read `metrics.json`, branch with `BranchPythonOperator`, load the warehouse in a `TaskGroup`, and get trigger rules right.
