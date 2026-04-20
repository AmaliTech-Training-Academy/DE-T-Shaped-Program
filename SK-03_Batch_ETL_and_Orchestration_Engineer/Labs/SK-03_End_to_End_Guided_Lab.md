# SK-03 END-TO-END GUIDED LAB: Building a Daily Sales Pipeline for UrbanGear

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **data engineer at UrbanGear**, a fast-growing online retailer processing 2 million daily orders across 3 platforms (website, mobile app, marketplace). The data team manually runs 12 SQL scripts every morning at 6 AM to prepare yesterday's sales data for the analytics team.

This process has collapsed:

- **An analyst runs the pipeline manually.** Last Tuesday they called in sick and nobody ran it. The CFO's 9 AM dashboard showed stale data from two days ago.
- **Silent failures.** Script #7 broke due to a source schema change. Nobody noticed for 3 days — by which time a $2M revenue gap had appeared in the weekly report.
- **No dependency enforcement.** Scripts must run in order (extract → clean → transform → aggregate → load) but there is nothing stopping someone from running them out of sequence.
- **Scaling issues.** As order volume grows, the scripts take longer. Last month they began missing the 9 AM deadline.
- **No data quality checks.** Negative quantities, future-dated orders, and null customer IDs flow straight into the BI dashboards.

**Your job:** Replace the manual process with a fully automated, orchestrated, monitored pipeline using Airflow + PySpark + dbt, with data quality checks, alerting, and self-healing retry logic.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why this problem?** | Manual pipelines that break silently are the most common data engineering problem on client projects. This is not a toy problem — it happens at every company that has outgrown scripts. |
| **Why PySpark + Airflow + dbt together?** | This is the real-world production stack. PySpark handles scale (millions of rows), Airflow handles orchestration and dependency management, dbt handles SQL transformation with testing and documentation. Each tool does what it is best at. |
| **When would you build this?** | Any time a client has: a manual pipeline, complex ETL dependencies, growing data volumes that outstrip SQL scripts, no monitoring, or repeated data quality incidents. |
| **What makes this different from a cron job?** | Cron runs a command — it has no concept of success or failure, no retry logic, no dependency management, and no UI to inspect what happened. Airflow manages all of these. |

---

## WHAT YOU WILL BUILD

```
UrbanGear Daily Sales Pipeline:
│
├── 1. EXTRACT LAYER (Airflow: PythonOperator + S3KeySensor)
│   ├── Check source data arrived in S3
│   └── Wait for all source files (up to 1 hour)
│
├── 2. TRANSFORM LAYER (PySpark / AWS Glue Job)
│   ├── Read raw JSON orders from S3 with explicit schema
│   ├── Tag each record with quality flags
│   ├── Route clean → S3 processed zone (partitioned Parquet)
│   ├── Route rejected → S3 dead-letter zone
│   └── Write quality metrics JSON sidecar
│
├── 3. QUALITY GATE (Airflow: BranchPythonOperator)
│   ├── Read quality metrics from S3
│   └── Branch: load if reject_rate < 5%, abort + alert if ≥ 5%
│
├── 4. LOAD LAYER (Airflow: TaskGroup → Redshift)
│   ├── DELETE today's partition from staging
│   └── COPY processed Parquet to Redshift staging
│
├── 5. TRANSFORMATION LAYER (dbt via BashOperator)
│   ├── stg_orders (staging model, incremental)
│   ├── fct_daily_sales (mart model, incremental)
│   └── dbt test (all schema.yml tests)
│
└── 6. ALERTING (Airflow: SlackWebhookOperator)
    ├── Success: pipeline summary with record counts
    └── Failure: task name, date, link to Airflow log
```

### Time Estimate: 8–10 hours

---

## PREREQUISITES

```bash
# Local setup
pip install pyspark==3.5.0 apache-airflow==2.8.0 dbt-redshift==1.7.0

# Docker Compose for Airflow (recommended)
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.8.0/docker-compose.yaml'
docker-compose up -d

# AWS CLI (for S3/Glue, if using the AWS variant)
pip install awscli boto3
```

---

## PART 1: PYSPARK ETL JOB (90 min)

### Step 1.1: Project Setup and Configuration

**WHY:** Externalizing all paths, thresholds, and business rules into a config class makes the job testable and environment-agnostic. Changing from dev to production is a config swap, not a code change.

```python
"""
UrbanGear Daily Sales ETL Job
==============================
PySpark job: reads raw JSON orders from S3, applies quality checks,
routes clean/rejected records, writes Parquet, and emits metrics.

Run locally:  python etl_job.py --process_date 2026-03-21
Run on Glue:  Deployed as a Glue job; date passed as script arg.
"""

import sys
import json
import logging
import argparse
from datetime import datetime, timedelta
from pathlib import Path

from pyspark.sql import SparkSession, DataFrame
from pyspark.sql import functions as F
from pyspark.sql.types import (
    StructType, StructField, StringType, IntegerType,
    DoubleType, TimestampType, BooleanType
)
from pyspark.sql.window import Window

# ── Logging ────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)


# ── Configuration ───────────────────────────────────────────────
class PipelineConfig:
    """
    All configuration lives here — paths, thresholds, business rules.
    Change environment by swapping this object, not the code.
    """
    # S3 paths (replace with your bucket)
    S3_RAW      = "s3://urbangear-data-lake/raw"
    S3_PROCESSED = "s3://urbangear-data-lake/processed"
    S3_REJECTED  = "s3://urbangear-data-lake/rejected"
    S3_METRICS   = "s3://urbangear-data-lake/metrics"

    # Quality thresholds
    MAX_NULL_RATE      = 0.02     # Max 2% nulls allowed in critical fields
    MAX_DUPLICATE_RATE = 0.001    # Max 0.1% duplicate order IDs
    MIN_RECORD_COUNT   = 100_000  # Expect at least 100K orders per day

    # Business rules
    VALID_PLATFORMS  = ["website", "mobile_app", "marketplace"]
    MAX_ORDER_VALUE  = 50_000.0   # Flag orders > $50K for review

    # Spark tuning
    SHUFFLE_PARTITIONS = 200
```

### Step 1.2: Schema Definition and SparkSession

**WHY:** Never let Spark infer schema from JSON in production. Inference reads the entire file, is slow, and is wrong for nullable fields (it will infer `phone_number` as a Long). Explicit schemas are your contract with the source system.

```python
# ── Schema ──────────────────────────────────────────────────────
RAW_ORDER_SCHEMA = StructType([
    StructField("order_id",        StringType(),    nullable=False),
    StructField("customer_id",     StringType(),    nullable=True),
    StructField("platform",        StringType(),    nullable=True),
    StructField("order_date",      TimestampType(), nullable=True),
    StructField("product_id",      StringType(),    nullable=True),
    StructField("product_name",    StringType(),    nullable=True),
    StructField("category",        StringType(),    nullable=True),
    StructField("quantity",        IntegerType(),   nullable=True),
    StructField("unit_price",      DoubleType(),    nullable=True),
    StructField("discount_pct",    DoubleType(),    nullable=True),
    StructField("shipping_cost",   DoubleType(),    nullable=True),
    StructField("tax_amount",      DoubleType(),    nullable=True),
    StructField("payment_method",  StringType(),    nullable=True),
    StructField("shipping_country",StringType(),    nullable=True),
    StructField("shipping_state",  StringType(),    nullable=True),
    StructField("shipping_city",   StringType(),    nullable=True),
    StructField("is_returned",     BooleanType(),   nullable=True),
])


def create_spark_session(app_name: str) -> SparkSession:
    """
    Create a SparkSession with production-tuned configuration.
    AQE is enabled: Spark will auto-coalesce shuffle partitions and
    auto-detect skew at runtime.
    """
    return (
        SparkSession.builder
        .appName(app_name)
        # Adaptive Query Execution — let Spark optimize at runtime
        .config("spark.sql.adaptive.enabled",                    "true")
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
        .config("spark.sql.shuffle.partitions",                  str(PipelineConfig.SHUFFLE_PARTITIONS))
        # Dynamic partition overwrite: only overwrite today's partition, not the whole table
        .config("spark.sql.sources.partitionOverwriteMode",      "dynamic")
        # Compression and serialization
        .config("spark.sql.parquet.compression.codec",           "snappy")
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
        .getOrCreate()
    )
```

### Step 1.3: Extract

```python
def extract_raw_data(spark: SparkSession, process_date: str) -> DataFrame:
    """
    Read raw order JSON from S3 for a specific date partition.

    WHY PERMISSIVE mode: We never want the job to crash on a single
    bad record. PERMISSIVE puts the bad row text in _corrupt_record
    so we can route it to the dead-letter zone.

    WHY fail on low record count: If only 1,000 records arrive when
    we expect 2M, it means the source export failed — not that we
    had a good day. Better to abort than to load misleading data.
    """
    raw_path = f"{PipelineConfig.S3_RAW}/orders/dt={process_date}/"
    logger.info(f"Reading raw data from: {raw_path}")

    df = (
        spark.read
        .schema(RAW_ORDER_SCHEMA)
        .option("mode",                  "PERMISSIVE")
        .option("columnNameOfCorruptRecord", "_corrupt_record")
        .json(raw_path)
    )

    count = df.count()
    logger.info(f"[EXTRACT] {count:,} records read")

    if count < PipelineConfig.MIN_RECORD_COUNT:
        raise ValueError(
            f"Record count {count:,} is below the minimum threshold "
            f"{PipelineConfig.MIN_RECORD_COUNT:,}. "
            "Source export may have failed. Aborting pipeline."
        )
    return df
```

### Step 1.4: Data Quality Tagging and Routing

**WHY:** Tagging quality problems as boolean columns before routing makes the logic auditable. The lineage of every rejected record is traceable — which rule rejected it, and why.

```python
def tag_and_route(df: DataFrame, process_date: str) -> tuple[DataFrame, DataFrame]:
    """
    Tag each record with quality flags and route to clean or rejected.

    Returns: (clean_df, rejected_df)

    The tagging approach (vs silent filter) means every rejected row
    has a documented reason. This is required for any regulated industry
    and makes incident investigation fast.
    """
    tagged = (
        df
        # Individual quality flags
        .withColumn("_null_critical",
            F.col("order_id").isNull()
            | F.col("customer_id").isNull()
            | F.col("order_date").isNull()
            | F.col("quantity").isNull()
            | F.col("unit_price").isNull()
        )
        .withColumn("_negative_qty",    F.col("quantity") <= 0)
        .withColumn("_future_date",     F.col("order_date") > F.current_timestamp())
        .withColumn("_invalid_platform",
            ~F.col("platform").isin(PipelineConfig.VALID_PLATFORMS)
        )
        .withColumn("_corrupt_json",    F.col("_corrupt_record").isNotNull())
        # Composite rejection flag
        .withColumn("_is_rejected",
            F.col("_null_critical")
            | F.col("_negative_qty")
            | F.col("_future_date")
            | F.col("_corrupt_json")
        )
        # Human-readable rejection reason
        .withColumn("rejection_reason",
            F.when(F.col("_corrupt_json"),    F.lit("corrupt_json"))
             .when(F.col("_null_critical"),   F.lit("null_critical_field"))
             .when(F.col("_negative_qty"),    F.lit("negative_quantity"))
             .when(F.col("_future_date"),     F.lit("future_order_date"))
             .otherwise(None)
        )
    )

    # Route: clean vs rejected
    rejected_df = tagged.filter(F.col("_is_rejected"))
    clean_df    = tagged.filter(~F.col("_is_rejected"))

    logger.info(f"[QUALITY] Clean: {clean_df.count():,} | Rejected: {rejected_df.count():,}")
    return clean_df, rejected_df


def transform_clean(df: DataFrame, process_date: str) -> DataFrame:
    """Apply business transformations to clean records."""
    window_order = Window.partitionBy("order_id")

    return (
        df
        # Calculated revenue measures
        .withColumn("gross_revenue",
            F.round(F.col("quantity") * F.col("unit_price"), 2))
        .withColumn("discount_amount",
            F.round(
                F.col("gross_revenue") * F.coalesce(F.col("discount_pct"), F.lit(0.0)) / 100,
                2
            ))
        .withColumn("net_revenue",
            F.round(F.col("gross_revenue") - F.col("discount_amount"), 2))
        .withColumn("total_amount",
            F.round(
                F.col("net_revenue")
                + F.coalesce(F.col("shipping_cost"), F.lit(0.0))
                + F.coalesce(F.col("tax_amount"),    F.lit(0.0)),
                2
            ))
        # Standardize text fields
        .withColumn("platform",          F.lower(F.trim(F.col("platform"))))
        .withColumn("payment_method",    F.upper(F.trim(F.col("payment_method"))))
        .withColumn("shipping_country",  F.upper(F.trim(F.col("shipping_country"))))
        # Date parts for partitioning
        .withColumn("order_year",  F.year(F.col("order_date")))
        .withColumn("order_month", F.month(F.col("order_date")))
        .withColumn("order_day",   F.dayofmonth(F.col("order_date")))
        .withColumn("order_hour",  F.hour(F.col("order_date")))
        .withColumn("day_of_week", F.dayofweek(F.col("order_date")))
        .withColumn("is_weekend",  F.col("day_of_week").isin(1, 7))
        # Order-level aggregates via window (no groupBy needed)
        .withColumn("order_item_count",
            F.count("*").over(window_order))
        .withColumn("order_total_revenue",
            F.sum("net_revenue").over(window_order))
        .withColumn("is_high_value",
            F.col("order_total_revenue") > PipelineConfig.MAX_ORDER_VALUE)
        # ETL metadata
        .withColumn("etl_processed_at",  F.current_timestamp())
        .withColumn("etl_process_date",  F.lit(process_date))
        # Drop internal flag columns
        .drop("_null_critical", "_negative_qty", "_future_date",
              "_invalid_platform", "_corrupt_json", "_is_rejected",
              "_corrupt_record")
    )
```

### Step 1.5: Quality Metrics and Write

```python
def collect_metrics(raw_df: DataFrame, clean_df: DataFrame,
                    rejected_df: DataFrame, process_date: str) -> dict:
    """
    Collect quality metrics and return as a dict.
    This dict is written to S3 as a JSON sidecar file and read by the
    Airflow quality gate task to decide whether to proceed or abort.
    """
    total    = raw_df.count()
    clean    = clean_df.count()
    rejected = rejected_df.count()

    # Per-rule rejection counts
    tagged = raw_df.select(
        F.col("order_id"),
        (F.col("order_id").isNull()
         | F.col("customer_id").isNull()
         | F.col("order_date").isNull()
         | F.col("quantity").isNull()
         | F.col("unit_price").isNull()).alias("null_critical"),
        (F.col("quantity") <= 0).alias("negative_qty"),
        (F.col("order_date") > F.current_timestamp()).alias("future_date"),
    )
    rule_counts = {
        row["rule"]: row["count"]
        for row in (
            tagged.select(
                F.sum(F.col("null_critical").cast("int")).alias("null_critical"),
                F.sum(F.col("negative_qty").cast("int")).alias("negative_qty"),
                F.sum(F.col("future_date").cast("int")).alias("future_date"),
            )
            .first()
            .asDict()
            .items()
        )
    } if tagged.count() > 0 else {}

    metrics = {
        "process_date":   process_date,
        "total_count":    total,
        "clean_count":    clean,
        "rejected_count": rejected,
        "reject_rate":    round(rejected / total, 4) if total > 0 else 0.0,
        "rule_counts":    rule_counts,
        "generated_at":   datetime.now().isoformat(),
    }
    logger.info(f"[METRICS] {metrics}")
    return metrics


def write_outputs(clean_df: DataFrame, rejected_df: DataFrame,
                  metrics: dict, process_date: str):
    """Write clean records, rejected records, and metrics to S3."""
    # Clean records: partitioned Parquet for efficient downstream reads
    (
        clean_df
        .repartition(F.col("order_year"), F.col("order_month"), F.col("order_day"))
        .write
        .mode("overwrite")
        .partitionBy("order_year", "order_month", "order_day")
        .parquet(f"{PipelineConfig.S3_PROCESSED}/orders/")
    )
    logger.info(f"[WRITE] Clean data written to {PipelineConfig.S3_PROCESSED}/orders/")

    # Rejected records: flat write for investigation
    if rejected_df.count() > 0:
        (
            rejected_df
            .coalesce(1)
            .write
            .mode("overwrite")
            .parquet(f"{PipelineConfig.S3_REJECTED}/orders/dt={process_date}/")
        )
        logger.info(f"[WRITE] {rejected_df.count():,} rejected records written to dead-letter")

    # Metrics sidecar (read by Airflow quality gate)
    metrics_path = f"{PipelineConfig.S3_METRICS}/orders/dt={process_date}/metrics.json"
    # Write via boto3 in production; for local testing write to disk
    import boto3, json as json_lib
    s3 = boto3.client("s3")
    bucket, key = metrics_path.replace("s3://", "").split("/", 1)
    s3.put_object(Bucket=bucket, Key=key, Body=json_lib.dumps(metrics))
    logger.info(f"[WRITE] Quality metrics written to {metrics_path}")
```

### Step 1.6: Main Entry Point

```python
def main(process_date: str = None):
    if process_date is None:
        process_date = (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d")

    logger.info("=" * 60)
    logger.info("URBANGEAR DAILY SALES ETL")
    logger.info(f"Processing date: {process_date}")
    logger.info(f"Started: {datetime.now().isoformat()}")
    logger.info("=" * 60)

    spark = create_spark_session("urbangear_daily_sales_etl")

    try:
        raw_df               = extract_raw_data(spark, process_date)
        clean_df, rejected_df = tag_and_route(raw_df, process_date)
        transformed_df        = transform_clean(clean_df, process_date)
        metrics               = collect_metrics(raw_df, transformed_df, rejected_df, process_date)
        write_outputs(transformed_df, rejected_df, metrics, process_date)

        logger.info("=" * 60)
        logger.info(f"ETL COMPLETE — {process_date}")
        logger.info(f"Clean: {metrics['clean_count']:,} | Rejected: {metrics['rejected_count']:,} | Rate: {metrics['reject_rate']:.2%}")
        logger.info("=" * 60)

    finally:
        spark.stop()


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--process_date", default=None)
    args = parser.parse_args()
    main(args.process_date)
```

---

## PART 2: AIRFLOW DAG (90 min)

### Step 2.1: DAG Setup and Callbacks

**WHY:** The default_args block defines the fallback behavior for every task in the DAG. Centralize retry policy and alerting here rather than repeating on every operator.

```python
"""
UrbanGear Daily Sales Pipeline — Airflow DAG
=============================================
Orchestrates: Check → Wait → Spark ETL → Quality Gate
             → [Load Redshift | Alert Failure] → dbt → Slack Success

Design principles:
- Idempotent: safe to re-run for any date
- catchup=False: no accidental historical backfill
- max_active_runs=1: prevents overlapping daily runs
- Every task has retries and execution_timeout
"""

from datetime import datetime, timedelta
import json

from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
from airflow.operators.empty import EmptyOperator
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
from airflow.utils.task_group import TaskGroup
from airflow.hooks.base import BaseHook

AWS_CONN_ID      = "aws_default"
REDSHIFT_CONN_ID = "redshift_default"
SLACK_CONN_ID    = "slack_webhook"
S3_BUCKET        = "urbangear-data-lake"


def _slack_failure_alert(context):
    """
    Callback: fires when any task fails.
    Sends a Slack message with task name, run date, and log URL.
    """
    task_instance = context["task_instance"]
    dag_run       = context["dag_run"]
    log_url       = task_instance.log_url

    message = (
        f":red_circle: *UrbanGear Pipeline FAILED*\n"
        f"*Task:* `{task_instance.task_id}`\n"
        f"*Date:* {dag_run.logical_date.strftime('%Y-%m-%d')}\n"
        f"*Log:* <{log_url}|View Task Log>"
    )
    SlackWebhookOperator(
        task_id="slack_failure_alert",
        slack_webhook_conn_id=SLACK_CONN_ID,
        message=message,
    ).execute(context=context)


DEFAULT_ARGS = {
    "owner":               "data-engineering",
    "depends_on_past":     False,
    "email_on_failure":    True,
    "email":               ["data-team@urbangear.com"],
    "email_on_retry":      False,
    "retries":             3,
    "retry_delay":         timedelta(minutes=10),
    "execution_timeout":   timedelta(hours=2),
    "on_failure_callback": _slack_failure_alert,
}
```

### Step 2.2: Python Callables

```python
def _check_source_data(**context):
    """
    Verify source files exist in S3 before starting the ETL.
    Prevents running the pipeline on empty data (silent failure).
    """
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook

    process_date = context["ds"]
    hook         = S3Hook(aws_conn_id=AWS_CONN_ID)
    prefix       = f"raw/orders/dt={process_date}/"
    keys         = hook.list_keys(bucket_name=S3_BUCKET, prefix=prefix)

    if not keys:
        raise FileNotFoundError(
            f"No source files at s3://{S3_BUCKET}/{prefix}. "
            "Source system export may not have completed."
        )

    context["ti"].xcom_push(key="source_file_count", value=len(keys))
    print(f"[CHECK] {len(keys)} source files found")
    return len(keys)


def _quality_gate(**context):
    """
    Read quality metrics written by the Spark job.
    Returns the task_id to execute next:
    - 'load_to_redshift.truncate_staging'  if quality passes
    - 'alert_quality_failure'              if quality fails
    """
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook

    process_date = context["ds"]
    hook         = S3Hook(aws_conn_id=AWS_CONN_ID)
    metrics_key  = f"metrics/orders/dt={process_date}/metrics.json"

    try:
        obj     = hook.get_key(metrics_key, bucket_name=S3_BUCKET)
        metrics = json.loads(obj.get()["Body"].read().decode())
    except Exception as e:
        raise RuntimeError(f"Could not read quality metrics: {e}")

    reject_rate = metrics.get("reject_rate", 1.0)
    context["ti"].xcom_push(key="reject_rate",    value=reject_rate)
    context["ti"].xcom_push(key="clean_count",    value=metrics.get("clean_count", 0))
    context["ti"].xcom_push(key="rejected_count", value=metrics.get("rejected_count", 0))

    if reject_rate > 0.05:
        print(f"[QUALITY GATE] FAILED — reject_rate={reject_rate:.2%} > 5% threshold")
        return "alert_quality_failure"

    print(f"[QUALITY GATE] PASSED — reject_rate={reject_rate:.2%}")
    return "load_to_redshift.truncate_staging"
```

### Step 2.3: DAG Definition and Task Dependencies

```python
with DAG(
    dag_id          = "urbangear_daily_sales_pipeline",
    default_args    = DEFAULT_ARGS,
    description     = "Daily sales: raw JSON → PySpark → Redshift → dbt → Slack",
    schedule        = "0 4 * * *",          # 4:00 AM UTC daily
    start_date      = datetime(2025, 1, 1),
    catchup         = False,                # Never backfill silently
    max_active_runs = 1,                    # Prevent day overlap
    tags            = ["production", "sales", "daily"],
) as dag:

    # ── 1. Check source data ─────────────────────────────────
    check_source = PythonOperator(
        task_id         = "check_source_data",
        python_callable = _check_source_data,
    )

    # ── 2. Sensor: wait up to 1 hour for all files ───────────
    # mode=reschedule frees the worker slot while waiting — critical
    # on small Airflow deployments with few worker slots.
    wait_for_files = S3KeySensor(
        task_id      = "wait_for_source_files",
        bucket_key   = "raw/orders/dt={{ ds }}/part-*.json",
        bucket_name  = S3_BUCKET,
        aws_conn_id  = AWS_CONN_ID,
        poke_interval = 300,    # Check every 5 minutes
        timeout       = 3600,   # Fail if not found within 1 hour
        mode          = "reschedule",
    )

    # ── 3. Spark ETL (AWS Glue job) ──────────────────────────
    # For Fabric variant: replace with MSFabricRunNotebookOperator
    run_spark_etl = GlueJobOperator(
        task_id              = "run_spark_etl",
        job_name             = "urbangear_daily_sales_etl",
        script_args          = {
            "--process_date":        "{{ ds }}",
            "--S3_RAW_BUCKET":       f"s3://{S3_BUCKET}/raw",
            "--S3_PROCESSED_BUCKET": f"s3://{S3_BUCKET}/processed",
        },
        aws_conn_id          = AWS_CONN_ID,
        region_name          = "us-east-1",
        num_of_dpus          = 10,
        wait_for_completion  = True,
        execution_timeout    = timedelta(hours=3),
    )

    # ── 4. Quality gate (branch) ─────────────────────────────
    quality_gate = BranchPythonOperator(
        task_id         = "quality_gate",
        python_callable = _quality_gate,
    )

    # ── 4a. Abort path: alert on quality failure ─────────────
    alert_quality_failure = SlackWebhookOperator(
        task_id               = "alert_quality_failure",
        slack_webhook_conn_id = SLACK_CONN_ID,
        message               = (
            ":x: *UrbanGear Pipeline ABORTED — Quality Gate Failed*\n"
            "*Date:* {{ ds }}\n"
            "*Reject Rate:* {{ ti.xcom_pull(task_ids='quality_gate', key='reject_rate') }}\n"
            "*Action:* Investigate s3://urbangear-data-lake/rejected/orders/dt={{ ds }}/"
        ),
    )

    # ── 4b. Load path: Redshift ──────────────────────────────
    with TaskGroup(group_id="load_to_redshift") as load_to_redshift:

        # DELETE today's data first (idempotent — safe to re-run)
        truncate_staging = SQLExecuteQueryOperator(
            task_id  = "truncate_staging",
            conn_id  = REDSHIFT_CONN_ID,
            sql      = """
                DELETE FROM staging.daily_orders
                WHERE order_date::date = '{{ ds }}'::date;
            """,
        )

        # COPY Parquet from S3 to Redshift
        copy_to_staging = S3ToRedshiftOperator(
            task_id          = "copy_to_staging",
            schema           = "staging",
            table            = "daily_orders",
            s3_bucket        = S3_BUCKET,
            s3_key           = (
                "processed/orders/"
                "order_year={{ macros.ds_format(ds, '%Y-%m-%d', '%Y') }}/"
                "order_month={{ macros.ds_format(ds, '%Y-%m-%d', '%-m') }}/"
                "order_day={{ macros.ds_format(ds, '%Y-%m-%d', '%-d') }}/"
            ),
            redshift_conn_id = REDSHIFT_CONN_ID,
            aws_conn_id      = AWS_CONN_ID,
            copy_options     = ["FORMAT AS PARQUET"],
            method           = "REPLACE",
        )

        truncate_staging >> copy_to_staging

    # ── 5. dbt transformations ───────────────────────────────
    run_dbt = BashOperator(
        task_id      = "run_dbt_transformations",
        bash_command = (
            "cd /opt/airflow/dbt/urbangear && "
            "dbt run  --select tag:daily --vars '{process_date: {{ ds }}}' "
            "         --profiles-dir /opt/airflow/dbt/profiles && "
            "dbt test --select tag:daily "
            "         --profiles-dir /opt/airflow/dbt/profiles"
        ),
        execution_timeout = timedelta(hours=1),
    )

    # ── 6. Success alert ─────────────────────────────────────
    # trigger_rule="none_failed_min_one_success" means this fires
    # even if the quality_failure branch was taken (skipped tasks
    # would otherwise block trigger_rule="all_success").
    alert_success = SlackWebhookOperator(
        task_id               = "alert_success",
        slack_webhook_conn_id = SLACK_CONN_ID,
        message               = (
            ":white_check_mark: *UrbanGear Daily Pipeline Complete*\n"
            "*Date:* {{ ds }}\n"
            "*Clean records:* {{ ti.xcom_pull(task_ids='quality_gate', key='clean_count') }}\n"
            "*Dashboard:* <https://bi.urbangear.com/sales|View Sales Dashboard>"
        ),
        trigger_rule = "none_failed_min_one_success",
    )

    # ── Task dependency graph ────────────────────────────────
    (
        check_source
        >> wait_for_files
        >> run_spark_etl
        >> quality_gate
        >> [alert_quality_failure, load_to_redshift]
    )
    load_to_redshift >> run_dbt >> alert_success
```

---

## PART 3: dbt TRANSFORMATION LAYER (60 min)

### Step 3.1: Staging Model

**WHY:** The staging layer is a 1:1 mirror of the source with types cleaned and names standardized. No business logic lives here. This keeps the mart models readable and makes source schema changes easy to isolate.

```sql
-- models/staging/stg_orders.sql
{{
    config(
        materialized = 'incremental',
        unique_key   = 'order_id',
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
        COALESCE(discount_pct,    0)                 AS discount_pct,
        COALESCE(shipping_cost,   0)                 AS shipping_cost,
        COALESCE(tax_amount,      0)                 AS tax_amount,
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

### Step 3.2: Mart Model

```sql
-- models/marts/fct_daily_sales.sql
{{
    config(
        materialized = 'incremental',
        unique_key   = 'order_id',
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

### Step 3.3: Schema Tests

```yaml
# models/marts/schema.yml
version: 2

models:
  - name: fct_daily_sales
    description: >
      Daily sales fact table — one row per order line item.
      Grain: one product purchased in one order.
      Populated by the UrbanGear daily Airflow pipeline.

    columns:
      - name: order_id
        description: Unique identifier for the order line item
        tests:
          - not_null
          - unique

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

---

## PART 4: VERIFICATION (20 min)

```sql
-- Run these checks after the pipeline completes to verify correctness.

-- 1. Record count for today's run
SELECT
    order_day,
    COUNT(*)                      AS line_item_count,
    COUNT(DISTINCT order_id)      AS order_count,
    COUNT(DISTINCT customer_id)   AS unique_customers,
    ROUND(SUM(net_revenue), 2)    AS total_net_revenue
FROM analytics.fct_daily_sales
WHERE order_day = CURRENT_DATE - 1
GROUP BY order_day;

-- 2. Platform distribution (should be stable day-over-day)
SELECT
    platform,
    COUNT(*) AS order_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM analytics.fct_daily_sales
WHERE order_day = CURRENT_DATE - 1
GROUP BY platform
ORDER BY order_count DESC;

-- 3. Verify no future-dated orders slipped through
SELECT COUNT(*) AS future_order_count
FROM analytics.fct_daily_sales
WHERE order_date > CURRENT_TIMESTAMP;
-- Expected: 0

-- 4. Verify no negative revenues
SELECT COUNT(*) AS negative_revenue_count
FROM analytics.fct_daily_sales
WHERE net_revenue < 0;
-- Expected: 0

-- 5. Verify idempotency: re-run the dbt model and confirm same count
-- (Run this after triggering a dbt full-refresh for today's date)
-- Count should be identical to query #1 above.
```

---

## REFLECTION QUESTIONS

1. The Airflow quality gate branches to `load_to_redshift.truncate_staging` (a task inside a TaskGroup) or `alert_quality_failure`. What would happen to `run_dbt` and `alert_success` if the `alert_quality_failure` branch is taken? Walk through the trigger_rule logic.

2. The Spark job uses `mode=PERMISSIVE` and writes corrupted records to a dead-letter path. Under what circumstance would you change this to `mode=FAILFAST`, and what would you need to add to the Airflow DAG to handle a FAILFAST failure gracefully?

3. The dbt staging model uses a `-3 day` buffer on the incremental watermark (`MAX(etl_processed_at) - INTERVAL '3 days'`). Why is this buffer necessary? What could go wrong without it?

4. The Glue job operator has `retries=3` inherited from `default_args`. The Glue job itself also has `retry_limit=2` set in the operator. If the Glue job fails, how many total attempts are made? Draw the retry sequence.

5. You need to add a new source (a 4th sales platform) that uploads its data as CSV instead of JSON. Walk through every file you would need to change: the PySpark job, the Airflow DAG, the dbt models, and the schema.yml tests.
