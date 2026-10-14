# Lab 05 — Airflow Fundamentals: Your First DAG

> **Goal:** Learn how Airflow runs work — DAGs, operators, dependencies, the logical date `{{ ds }}`, retries, timeouts, XCom and failure callbacks — on a small teaching DAG before building the real pipeline.
> **Time:** 150–210 min | **Difficulty:** Beginner | **Prerequisites:** Lab 04
> **Guided lab coverage:** Part 2 Step 2.1 (imports, `default_args`: `owner`, `retries`, `retry_delay`, `execution_timeout`, `on_failure_callback`; `catchup=False`), Step 2.2 (Python callables reading `context["ds"]`, XCom) and Step 2.3 (`with DAG(...)`, `schedule`, `>>` dependencies, `EmptyOperator`)

## Learning objectives

By the end of this lab you can:

- Explain the scheduler, webserver and metadata database, and why a DAG file is a *description*, not a script.
- Explain the **data interval**: why the run for 2026-07-01 executes on 2026-07-02 and has `ds = 2026-07-01`.
- Use `BashOperator`, `PythonOperator` and `EmptyOperator`, and wire them with `>>`.
- Configure retries, `execution_timeout` and an `on_failure_callback` through `default_args`.
- Pass a value between tasks with **XCom**.
- Test a DAG from the command line, read task logs, and recover a failed task with **Clear**.

## Steps

### Step 1 — Know how Airflow thinks

- **Scheduler** — parses the files in `dags\`, decides which runs are due, queues tasks, enforces retries.
- **Webserver** — the UI on http://localhost:8080.
- **Metadata database** (`airflow-db`) — every run, task state, connection and XCom lives here.
- A **DAG** (Directed Acyclic Graph) is a set of **tasks** plus the order between them. Tasks are instances of **operators** (`BashOperator` runs a shell command, `PythonOperator` calls a Python function).
- **The DAG file is a description.** The scheduler re-imports it every ~30 seconds, so the top level of the file must be cheap — real work happens *inside* tasks.

**The data interval.** A daily DAG's run "for 2026-07-01" covers the data of 2026-07-01 and starts **after** that day ends (early on 2026-07-02). Inside the run, the template `{{ ds }}` (Python: `context["ds"]`) is `2026-07-01` — the start of the interval. That is exactly the `--process_date` your `etl_job.py` wants.

**Expected output:** none.

✅ **Checkpoint:** you can answer "a daily run executes on Tuesday — which day's data does it process?" (Monday's.)

### Step 2 — Write the teaching DAG

Create `C:\DataEngineering\sk03\dags\urbangear_hello.py`:

```python
"""Lab 05: teaching DAG — operators, dependencies, {{ ds }}, retries, XCom, alert callback."""
import os
from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import PythonOperator


def _alert(context):
    """on_failure_callback: runs once a task has failed its LAST retry."""
    ti = context["task_instance"]
    print(f"[ALERT] task={ti.task_id} date={context['ds']} FAILED. Log: {ti.log_url}")


default_args = {
    "owner":               "data-engineering",
    "retries":             2,                      # 3 attempts in total
    "retry_delay":         timedelta(minutes=1),
    "execution_timeout":   timedelta(minutes=10),  # kill a task that hangs
    "on_failure_callback": _alert,
}


def _count_source_files(**context):
    """List the day's raw files. The RETURN value is stored as an XCom."""
    import boto3

    s3 = boto3.client(
        "s3",
        endpoint_url=os.environ["MINIO_ENDPOINT"],
        aws_access_key_id=os.environ["MINIO_ACCESS_KEY"],
        aws_secret_access_key=os.environ["MINIO_SECRET_KEY"],
    )
    prefix = f"raw/orders/dt={context['ds']}/"
    n = len(s3.list_objects_v2(Bucket="urbangear-data-lake", Prefix=prefix).get("Contents", []))
    print(f"[COUNT] {n} file(s) under {prefix}")
    return n


def _report(**context):
    """Read the upstream task's XCom."""
    n = context["ti"].xcom_pull(task_ids="count_source_files")
    print(f"[REPORT] date={context['ds']} files={n}")


with DAG(
    dag_id="urbangear_hello",
    description="Lab 05 teaching DAG",
    start_date=datetime(2026, 7, 1),
    schedule="@daily",
    catchup=False,                     # never create runs for the past automatically
    default_args=default_args,
    tags=["lab05"],
) as dag:

    start = EmptyOperator(task_id="start")      # does nothing; a clear visual start node

    check_minio = BashOperator(
        task_id="check_minio",
        bash_command="curl -sf http://minio:9000/minio/health/live && echo MINIO_OK",
    )
    print_date = BashOperator(
        task_id="print_logical_date",
        bash_command="echo 'processing data interval starting {{ ds }}'",
    )
    count_files = PythonOperator(task_id="count_source_files", python_callable=_count_source_files)
    report = PythonOperator(task_id="report", python_callable=_report)

    start >> [check_minio, print_date] >> count_files >> report
```

- **`default_args`** apply to every task: 2 retries one minute apart, a 10-minute timeout, and the `_alert` callback — the same keys the guided lab's `DEFAULT_ARGS` uses.
- **`start >> [check_minio, print_date] >> count_files >> report`** — `start` first, then two tasks in parallel, then both must succeed before `count_source_files` (a *fan-in*).
- **`{{ ds }}`** in `bash_command` is rendered by Jinja per run. In a Python function you read `context["ds"]` instead — templates only render in operator arguments.
- **XCom** ("cross-communication"): a `PythonOperator`'s return value is saved; another task reads it with `ti.xcom_pull(task_ids=...)`. Use XCom for *small* values (counts, paths) — never for datasets.
- **`EmptyOperator`** does nothing; it is used as a start/end/join node to make graphs clearer (Lab 07 uses one to join two branches).

**Expected output:** none yet.

✅ **Checkpoint:** the file is saved in `C:\DataEngineering\sk03\dags\`.

### Step 3 — Check the DAG imports cleanly

```powershell
docker exec -it airflow-scheduler airflow dags list-import-errors
docker exec -it airflow-scheduler airflow dags list
```

**Expected output:** the first command prints `No data found`. The second prints a table (layout approximate) with a row like

```
dag_id          | fileloc                              | owners           | is_paused
================+======================================+==================+==========
urbangear_hello | /opt/airflow/dags/urbangear_hello.py | data-engineering | True
```

If `list-import-errors` shows a traceback, fix the Python error it names and run it again. New DAGs start **paused** (`is_paused True`).

✅ **Checkpoint:** no import errors, and `urbangear_hello` is listed.

### Step 4 — Run the whole DAG once from the command line

`airflow dags test` runs every task of one run in a single process, in dependency order, and prints all logs — the fastest way to try a DAG without the UI or the scheduler.

```powershell
docker exec -it airflow-scheduler airflow dags test urbangear_hello 2026-07-01
```

**Expected output** (many log lines; find these):

```
... MINIO_OK
... processing data interval starting 2026-07-01
... [COUNT] 1 file(s) under raw/orders/dt=2026-07-01/
... [REPORT] date=2026-07-01 files=1
... Marking task as SUCCESS. dag_id=urbangear_hello, task_id=report, ...
... DagRun Finished: dag_id=urbangear_hello, ... state=success, ...
```

The logical date you passed (`2026-07-01`) became `{{ ds }}` everywhere, and `report` read the `1` that `count_source_files` returned via XCom.

✅ **Checkpoint:** `[REPORT] date=2026-07-01 files=1` and `state=success`.

### Step 5 — Trigger it in the UI and read the logs

1. Open http://localhost:8080 → `urbangear_hello`. Toggle it **on** (switch left of the name).
2. Click **▶ Trigger DAG w/ config**, set **Logical date** to `2026-07-02 00:00:00`, click **Trigger**.
3. Open the **Grid** view: rows are tasks, columns are runs. Squares go light green (running) → dark green (success).
4. Click the `report` square → **Logs**. Then click `count_source_files` → **XCom**.

**Expected output:** the `report` log contains `[REPORT] date=2026-07-02 files=1`; the XCom tab of `count_source_files` shows key `return_value`, value `1`. The **Graph** tab shows `start` fanning out to two tasks that fan back in.

(Turning the DAG on also makes the scheduler create one run for the most recent full day — because `catchup=False`, it creates only that one, not one per day since `start_date`. Its `count_source_files` logs `0 file(s)` because there is no data for that date.)

✅ **Checkpoint:** you found `files=1` in the log and `return_value 1` in XCom.

### Step 6 — Break a task: watch retries and the failure alert

Change `check_minio`'s command in the DAG file to a port where nothing listens:

```python
        bash_command="curl -sf http://minio:9999/minio/health/live && echo MINIO_OK",
```

Save, wait ~30 seconds, then trigger again with logical date `2026-07-03 00:00:00`.

**Expected output:** in Grid view `check_minio` turns **yellow** (`up_for_retry`) twice, about a minute apart, then **red** (`failed`). `count_source_files` and `report` become **orange** (`upstream_failed`) — they never ran. Open `check_minio` → **Logs**, use the attempt selector (1, 2, 3); the log of attempt 3 ends with a line like

```
[ALERT] task=check_minio date=2026-07-03 FAILED. Log: http://localhost:8080/...
```

In production, that callback would post to Slack or PagerDuty instead of printing (the guided lab's `_slack_failure_alert`; Lab 06 builds ours).

Now fix the port back to `9000`, save, wait ~30 seconds, click the red `check_minio` square → **Clear** → confirm. Clearing means "forget this result and run it again" — it re-runs `check_minio` *and* its downstream tasks.

**Expected output:** all squares of the 2026-07-03 run turn dark green.

✅ **Checkpoint:** you saw 3 attempts, the `[ALERT]` line, and a successful recovery with **Clear**.

### Step 7 — See when the schedule fires

```powershell
docker exec -it airflow-scheduler airflow dags next-execution urbangear_hello
```

**Expected output:** a timestamp at **00:00:00+00:00** — the logical date of the next scheduled run. That run will actually *start* one day later, when its interval has closed.

The guided lab's DAG uses `schedule = "0 4 * * *"` (cron: minute 0, hour 4, every day) — runs start at 04:00 UTC, giving the source system time to deliver files.

Finally, toggle `urbangear_hello` **off** so it doesn't keep running every day.

✅ **Checkpoint:** you can explain the difference between a run's logical date and the moment it starts.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| DAG doesn't appear in the UI | Import error, or saved outside `dags\` | `airflow dags list-import-errors`; check the folder |
| "Broken DAG" banner in the UI | Python error in the file | Read the traceback in the banner, fix, save |
| Tasks stay grey / "queued" | DAG is paused, or scheduler down | Toggle the DAG on; `docker compose ps` — is `airflow-scheduler` Up? |
| `{{ ds }}` printed literally | Used inside a Python function | Templates render only in operator arguments; use `context["ds"]` in Python |
| `xcom_pull` returns `None` | Wrong `task_ids`, or upstream returned nothing | Use the exact upstream `task_id`; make the callable `return` the value |
| `[COUNT] 0 file(s)` | No raw data for that logical date | Trigger with a date that has data (2026-07-01 … 07-05) |
| Login fails | `airflow-init` failed | `docker compose logs airflow-init` |

## Key takeaways

- A DAG file describes tasks and their order; the scheduler runs them.
- The run for date D executes after D ends, and `{{ ds }}` / `context["ds"]` = D — pass it to jobs as `--process_date`.
- `default_args` sets retries, `retry_delay`, `execution_timeout` and `on_failure_callback` for every task.
- XCom passes *small* values between tasks; `EmptyOperator` is a structural node.
- `airflow dags test <dag> <date>` is the fastest way to try a DAG; **Clear** is the standard recovery after a fix.

## Next

[Lab 06 — Sensors, Hooks and the Spark Task](Lab_06_Sensors_Hooks_and_the_Spark_Task.md): start the real `urbangear_daily_sales_pipeline` DAG — wait for the files, check them with an S3 hook, and run `etl_job.py` for `{{ ds }}`.
