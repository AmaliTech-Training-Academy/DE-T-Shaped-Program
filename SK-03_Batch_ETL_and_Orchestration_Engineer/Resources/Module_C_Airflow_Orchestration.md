# Module C: Workflow Orchestration with Apache Airflow

**Time:** 8-10 hours | **Focus:** DAGs, operators, scheduling, production patterns

---

## C.1 Airflow Fundamentals

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Astronomer — Introduction to Airflow](https://docs.astronomer.io/learn/intro-to-airflow) | 30 min | Best free Airflow learning content |
| [Airflow Core Concepts — Official Docs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html) | 45 min | DAGs, Tasks, Operators, XComs |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Apache Airflow Full Tutorial — Marc Lamberti](https://www.youtube.com/watch?v=K9AnJ9_ZAXE) | 2 hrs | Best single Airflow tutorial |

---

## C.2 DAG Design Patterns

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Idempotency in Airflow DAGs — Maxime Beauchemin](https://maximebeauchemin.medium.com/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a) | 25 min | Why idempotency is critical, by Airflow's creator |
| [TaskFlow API — Official Docs](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html) | 20 min | Modern @task decorator approach |

### DAG Design Checklist

```
BEFORE DEPLOYING ANY DAG:

Structure:
□ dag_id is unique and descriptive (domain_pipeline_frequency)
□ start_date is a static datetime, NOT datetime.now()
□ catchup=False (unless backfill is intentional)
□ max_active_runs=1 (prevents overlap)

Tasks:
□ Every task has retries and retry_delay configured
□ Long tasks have execution_timeout
□ on_failure_callback is set for alerts
□ SLA is set on critical tasks

Design:
□ All tasks are idempotent (re-run safe)
□ XCom values are small metadata only
□ Sensors use mode=reschedule (not poke)
□ Credentials use conn_id, never hardcoded
□ Templates use {{ ds }} not hardcoded dates
```

---

## C.3 Trigger Rules

### Reference

```
trigger_rule controls WHEN a task runs based on upstream state.

"all_success"    → Run if ALL upstream succeeded (default)
"all_done"       → Run regardless of upstream state
"one_success"    → Run if AT LEAST ONE succeeded
"none_failed"    → Run if NO upstream failed (skips OK)

COMMON PATTERN — Quality gate with branch:
    quality_gate (BranchPythonOperator)
        ↓ branch A         ↓ branch B
    alert_failure      load_warehouse
        ↓ SKIPPED            ↓ SUCCESS
              ↓         ↓
           final_alert (trigger_rule="none_failed")
```

---

## C.4 Sensors & Advanced Patterns

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Sensors in Airflow — Astronomer](https://docs.astronomer.io/learn/what-is-a-sensor) | 20 min | All sensor types, poke vs reschedule |

### Template: Production DAG

```python
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime, timedelta

default_args = {
    "retries": 3,
    "retry_delay": timedelta(minutes=10),
    "execution_timeout": timedelta(hours=2),
}

with DAG(
    dag_id="sales_etl_daily",
    start_date=datetime(2026, 1, 1),
    schedule_interval="0 6 * * *",
    catchup=False,
    max_active_runs=1,
    default_args=default_args,
    tags=["etl", "sales"],
) as dag:

    extract = PythonOperator(
        task_id="extract",
        python_callable=extract_fn,
    )

    with TaskGroup("transform") as transform_group:
        clean = PythonOperator(task_id="clean", python_callable=clean_fn)
        enrich = PythonOperator(task_id="enrich", python_callable=enrich_fn)
        clean >> enrich

    quality_check = BranchPythonOperator(
        task_id="quality_check",
        python_callable=check_quality,
    )

    load = PythonOperator(task_id="load", python_callable=load_fn)
    alert = PythonOperator(task_id="alert", python_callable=send_alert)

    extract >> transform_group >> quality_check
    quality_check >> [load, alert]
```

---

## Checkpoint

Before moving to Module D, you should be able to:

- [ ] Write a DAG with proper configuration
- [ ] Use TaskGroups to organize complex workflows
- [ ] Implement branching with BranchPythonOperator
- [ ] Configure sensors for external dependencies
