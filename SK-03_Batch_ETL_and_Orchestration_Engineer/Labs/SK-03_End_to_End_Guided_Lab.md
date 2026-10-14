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

You build and run everything on the **local stack** from Granular Lab 00 (MinIO = S3, Postgres = Redshift, PySpark in Airflow = Glue, logged alerts = Slack). Blocks marked **"On AWS (preview of SK-04)"** show the production equivalents.

```
UrbanGear Daily Sales Pipeline:
│
├── 1. EXTRACT LAYER (Airflow: S3KeySensor + PythonOperator with S3Hook)
│   ├── Wait for the day's source files (up to 1 hour)
│   └── Check and count the source files (XCom)
│
├── 2. TRANSFORM LAYER (PySpark — local spark-submit; AWS Glue in production)
│   ├── Read raw JSON orders with an explicit schema (PERMISSIVE + _corrupt_record)
│   ├── Tag each record with quality flags
│   ├── Route clean → processed zone (partitioned Parquet, dynamic overwrite)
│   ├── Route rejected → dead-letter zone
│   └── Write quality metrics JSON sidecar
│
├── 3. QUALITY GATE (Airflow: BranchPythonOperator)
│   ├── Read quality metrics from the data lake
│   └── Branch: load if reject_rate ≤ 5%, abort + alert if > 5%
│
├── 4. LOAD LAYER (Airflow: TaskGroup → warehouse; Redshift in production)
│   ├── DELETE this date's rows from staging
│   └── COPY the date's processed Parquet into staging
│
├── 5. TRANSFORMATION LAYER (dbt via BashOperator)
│   ├── stg_orders (staging model, incremental)
│   ├── fct_daily_sales (mart model, incremental)
│   └── dbt test (all schema.yml tests + a singular test)
│
└── 6. ALERTING (failure callback + alert tasks; Slack webhook optional)
    ├── Success: pipeline summary with record counts
    └── Failure: task name, date, link to Airflow log
```

### Time Estimate: 8–10 hours

---

## PREREQUISITES

**1. The local platform.** Complete [Granular Lab 00 — Environment Setup](../Granular%20Labs/Lab_00_Environment_Setup.md). It gives you, in `C:\DataEngineering\sk03`, one Docker Compose stack with everything this lab uses:

| Component | Version | Container / access |
|---|---|---|
| Apache Airflow (LocalExecutor) | 2.9.3 (Python 3.11) | `airflow-webserver` http://localhost:8080 (`admin`/`admin`), `airflow-scheduler` |
| PySpark (+ S3A JARs, Java 17) | 3.5.1 | inside the Airflow image (`spark-submit`) |
| dbt Core + dbt-postgres | 1.8.x | inside the Airflow image |
| MinIO (S3 API) | `bitnamilegacy/minio:2025.4.22` | http://localhost:9001 (`minioadmin`/`minioadmin123`); code: `http://minio:9000` |
| Spark standalone cluster | `bitnamilegacy/spark:3.5` | http://localhost:8081 (used for learning only) |
| Postgres warehouse | 16 | `warehouse` — Windows `localhost:5433`, code `warehouse:5432` (`dwh_user`/`dwh_pass`, db `warehouse`) |

> **Image note:** MinIO no longer publishes images on Docker Hub, and Bitnami moved its free images to `bitnamilegacy/*` in 2025. These images are **frozen** (no updates or security patches). We pin them because they exist and keep the lab reproducible — fine for a local lab, never for production.

**2. Skills.** Granular Labs 01–08 teach every technique used here, in this lab's order. If a step below feels unfamiliar, the matching granular lab is listed in its heading notes.

**3. Data and connections** (all from PowerShell in `C:\DataEngineering\sk03`; Granular Labs 01, 06, 07 explain them):

```powershell
docker compose up -d
# bucket + five days of raw JSON orders (2026-07-05 is a deliberately bad day)
docker exec -it airflow-scheduler python /opt/airflow/jobs/setup_buckets.py
foreach ($d in "2026-07-01", "2026-07-02", "2026-07-03", "2026-07-04") {
  docker exec airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date $d
}
docker exec airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-05 --dirty_rate 0.12
# Airflow connections: MinIO as "aws_default", the Postgres warehouse as "warehouse_default"
docker exec -it airflow-scheduler airflow connections add aws_default --conn-uri "aws://minioadmin:minioadmin123@/?endpoint_url=http%3A%2F%2Fminio%3A9000"
docker exec -it airflow-scheduler airflow connections add warehouse_default --conn-type postgres --conn-host warehouse --conn-port 5432 --conn-login dwh_user --conn-password dwh_pass --conn-schema warehouse
```

(`setup_buckets.py` and `generate_orders.py` are listed in Granular Lab 01.) The generator writes one JSON Lines file per day to `s3://urbangear-data-lake/raw/orders/dt=<date>/part-00000.json`: about 2,000 order lines, ~3% of them deliberately dirty.

---

## PART 1: PYSPARK ETL JOB (90 min)

All of Part 1 is one file: `C:\DataEngineering\sk03\jobs\etl_job.py` (Granular Labs 02–04 build it step by step).

### Step 1.1: Project Setup and Configuration

**WHY:** Externalizing all paths, thresholds, and business rules into a config class makes the job testable and environment-agnostic. Changing from dev to production is a config swap, not a code change.

```python
"""
UrbanGear Daily Sales ETL Job
==============================
PySpark job: reads raw JSON orders from the data lake, applies quality checks,
routes clean/rejected records, writes Parquet, and emits metrics.

Run locally (inside the Airflow container):
    spark-submit /opt/airflow/jobs/etl_job.py --process_date 2026-07-01
On AWS Glue: deployed as a Glue job; the date is passed as a job argument.
"""

import argparse
import json
import logging
import os
from datetime import datetime

import boto3
from pyspark.sql import DataFrame, SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import (
    BooleanType, DoubleType, IntegerType, StringType, StructField, StructType,
    TimestampType,
)
from pyspark.sql.window import Window

# ── Logging ────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)


# ── Configuration ───────────────────────────────────────────────
class PipelineConfig:
    """
    All configuration lives here — paths, thresholds, business rules.
    Change environment by swapping this object, not the code.
    """
    # Data-lake paths. s3a:// is Spark's S3 connector (MinIO locally, S3 on AWS).
    S3_BUCKET    = "urbangear-data-lake"
    S3_RAW       = f"s3a://{S3_BUCKET}/raw"
    S3_PROCESSED = f"s3a://{S3_BUCKET}/processed"
    S3_REJECTED  = f"s3a://{S3_BUCKET}/rejected"
    S3_METRICS   = f"s3a://{S3_BUCKET}/metrics"

    # Quality thresholds
    MAX_REJECT_RATE  = 0.05     # Airflow's quality gate stops the load above 5%
    MIN_RECORD_COUNT = 1_000    # local sample is ~2,000 lines/day (production: 100_000)

    # Business rules
    VALID_PLATFORMS = ["website", "mobile_app", "marketplace"]
    MAX_ORDER_VALUE = 50_000.0  # flag orders > $50K for review

    # Spark tuning
    SHUFFLE_PARTITIONS = 8      # small local data (production: 200)
```

### Step 1.2: Schema Definition and SparkSession

**WHY:** Never let Spark infer schema from JSON in production. Inference reads the entire file, is slow, and is wrong for nullable fields (it will infer `phone_number` as a Long). Explicit schemas are your contract with the source system. In PERMISSIVE mode the `_corrupt_record` column **must** be part of that schema, or Spark drops corrupt lines' text.

```python
# ── Schema ──────────────────────────────────────────────────────
RAW_ORDER_SCHEMA = StructType([
    StructField("order_id",         StringType(),    nullable=False),
    StructField("customer_id",      StringType(),    nullable=True),
    StructField("platform",         StringType(),    nullable=True),
    StructField("order_date",       TimestampType(), nullable=True),
    StructField("product_id",       StringType(),    nullable=True),
    StructField("product_name",     StringType(),    nullable=True),
    StructField("category",         StringType(),    nullable=True),
    StructField("quantity",         IntegerType(),   nullable=True),
    StructField("unit_price",       DoubleType(),    nullable=True),
    StructField("discount_pct",     DoubleType(),    nullable=True),
    StructField("shipping_cost",    DoubleType(),    nullable=True),
    StructField("tax_amount",       DoubleType(),    nullable=True),
    StructField("payment_method",   StringType(),    nullable=True),
    StructField("shipping_country", StringType(),    nullable=True),
    StructField("shipping_state",   StringType(),    nullable=True),
    StructField("shipping_city",    StringType(),    nullable=True),
    StructField("is_returned",      BooleanType(),   nullable=True),
    # PERMISSIVE mode puts the raw text of unparseable lines here.
    # It MUST be declared in an explicit schema, or Spark silently drops it.
    StructField("_corrupt_record",  StringType(),    nullable=True),
])


def create_spark_session(app_name: str) -> SparkSession:
    """
    Create a SparkSession with production-tuned configuration.
    AQE is enabled: Spark will auto-coalesce shuffle partitions and
    auto-detect skew at runtime.
    """
    builder = (
        SparkSession.builder
        .appName(app_name)
        # Adaptive Query Execution — let Spark optimize at runtime
        .config("spark.sql.adaptive.enabled",                    "true")
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
        .config("spark.sql.shuffle.partitions",                  str(PipelineConfig.SHUFFLE_PARTITIONS))
        # Dynamic partition overwrite: only overwrite the partitions we write, not the whole table
        .config("spark.sql.sources.partitionOverwriteMode",      "dynamic")
        # Compression and serialization
        .config("spark.sql.parquet.compression.codec",           "snappy")
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
        .config("spark.sql.session.timeZone",                    "UTC")
    )
    # Local stack: point the s3a connector at MinIO. (On AWS Glue/EMR these
    # env vars are absent and Spark uses the real S3 endpoint + IAM role.)
    if os.environ.get("MINIO_ENDPOINT"):
        builder = (
            builder
            .config("spark.hadoop.fs.s3a.endpoint",          os.environ["MINIO_ENDPOINT"])
            .config("spark.hadoop.fs.s3a.access.key",        os.environ["MINIO_ACCESS_KEY"])
            .config("spark.hadoop.fs.s3a.secret.key",        os.environ["MINIO_SECRET_KEY"])
            .config("spark.hadoop.fs.s3a.path.style.access", "true")
        )
    return builder.getOrCreate()
```

> **On AWS (preview of SK-04):** in an AWS Glue job, Glue provides the SparkSession and S3 access through the job's IAM role, `MINIO_ENDPOINT` is not set, and paths can use `s3://`. The rest of the job is unchanged.

### Step 1.3: Extract

```python
# ── Extract ─────────────────────────────────────────────────────
def extract_raw_data(spark: SparkSession, process_date: str) -> DataFrame:
    """
    Read raw order JSON for one date partition.

    WHY PERMISSIVE mode: we never want the job to crash on a single bad
    record. PERMISSIVE puts the bad line's text in _corrupt_record so we
    can route it to the dead-letter zone.

    WHY fail on low record count: if far fewer records arrive than expected,
    the source export probably failed — not a quiet day. Better to abort
    than to load misleading data.
    """
    raw_path = f"{PipelineConfig.S3_RAW}/orders/dt={process_date}/"
    logger.info(f"Reading raw data from: {raw_path}")

    df = (
        spark.read
        .schema(RAW_ORDER_SCHEMA)
        .option("mode",                      "PERMISSIVE")
        .option("columnNameOfCorruptRecord", "_corrupt_record")
        .json(raw_path)
        .cache()   # Spark refuses some queries on _corrupt_record unless the data is cached
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

**WHY:** Tagging quality problems as boolean columns before routing makes the logic auditable. The lineage of every rejected record is traceable — which rule rejected it, and why. Every flag is NULL-safe (`coalesce`), so every row lands in exactly one output.

```python
# ── Quality tagging and routing ─────────────────────────────────
def tag_and_route(df: DataFrame, process_date: str) -> tuple[DataFrame, DataFrame]:
    """
    Tag each record with quality flags and route to clean or rejected.

    Returns: (clean_df, rejected_df)

    Every flag is wrapped in coalesce(..., False/True) so it is never NULL.
    (In SQL, `NULL OR FALSE` is NULL, and a NULL filter keeps the row in
    NEITHER output — it would silently vanish.)
    """
    platform_clean = F.lower(F.trim(F.col("platform")))
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
        .withColumn("_negative_qty",
            F.coalesce(F.col("quantity") <= 0, F.lit(False)))
        .withColumn("_future_date",
            F.coalesce(F.col("order_date") > F.current_timestamp(), F.lit(False)))
        .withColumn("_invalid_platform",            # missing platform counts as invalid
            F.coalesce(~platform_clean.isin(PipelineConfig.VALID_PLATFORMS), F.lit(True)))
        .withColumn("_corrupt_json", F.col("_corrupt_record").isNotNull())
        # Composite rejection flag
        .withColumn("_is_rejected",
            F.col("_null_critical")
            | F.col("_negative_qty")
            | F.col("_future_date")
            | F.col("_invalid_platform")
            | F.col("_corrupt_json")
        )
        # Human-readable rejection reason (first matching rule wins)
        .withColumn("rejection_reason",
            F.when(F.col("_corrupt_json"),     F.lit("corrupt_json"))
             .when(F.col("_null_critical"),    F.lit("null_critical_field"))
             .when(F.col("_negative_qty"),     F.lit("negative_quantity"))
             .when(F.col("_future_date"),      F.lit("future_order_date"))
             .when(F.col("_invalid_platform"), F.lit("invalid_platform"))
             .otherwise(None)
        )
    )

    # Route: clean vs rejected
    rejected_df = tagged.filter(F.col("_is_rejected"))
    clean_df    = tagged.filter(~F.col("_is_rejected"))

    logger.info(f"[QUALITY] Clean: {clean_df.count():,} | Rejected: {rejected_df.count():,}")
    return clean_df, rejected_df


# ── Transform ───────────────────────────────────────────────────
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
            F.round(F.sum("net_revenue").over(window_order), 2))
        .withColumn("is_high_value",
            F.col("order_total_revenue") > PipelineConfig.MAX_ORDER_VALUE)
        # ETL metadata
        .withColumn("etl_processed_at",  F.current_timestamp())
        .withColumn("etl_process_date",  F.lit(process_date).cast("date"))
        # Drop internal flag columns
        .drop("_null_critical", "_negative_qty", "_future_date",
              "_invalid_platform", "_corrupt_json", "_is_rejected",
              "_corrupt_record", "rejection_reason")
    )
```

### Step 1.5: Quality Metrics and Write

```python
# ── Metrics and write ───────────────────────────────────────────
def collect_metrics(raw_df: DataFrame, clean_df: DataFrame,
                    rejected_df: DataFrame, process_date: str) -> dict:
    """
    Collect quality metrics and return them as a dict.
    This dict is written as a JSON "sidecar" file and read by the
    Airflow quality gate task to decide whether to proceed or abort.
    """
    total    = raw_df.count()
    clean    = clean_df.count()
    rejected = rejected_df.count()

    # Per-rule rejection counts (each rejected row counted once, by its reason)
    rule_counts = {
        row["rejection_reason"]: row["count"]
        for row in rejected_df.groupBy("rejection_reason").count().collect()
    }

    metrics = {
        "process_date":   process_date,
        "total_count":    total,
        "clean_count":    clean,
        "rejected_count": rejected,
        "reject_rate":    round(rejected / total, 4) if total > 0 else 0.0,
        "rule_counts":    dict(sorted(rule_counts.items())),
        "generated_at":   datetime.now().isoformat(),
    }
    logger.info(f"[METRICS] {metrics}")
    return metrics


def write_outputs(clean_df: DataFrame, rejected_df: DataFrame,
                  metrics: dict, process_date: str) -> None:
    """Write clean records, rejected records, and metrics to the data lake."""
    # Clean records: partitioned Parquet. With partitionOverwriteMode=dynamic,
    # "overwrite" replaces ONLY the partitions present in clean_df.
    (
        clean_df
        .repartition(F.col("order_year"), F.col("order_month"), F.col("order_day"))
        .write
        .mode("overwrite")
        .partitionBy("order_year", "order_month", "order_day")
        .parquet(f"{PipelineConfig.S3_PROCESSED}/orders/")
    )
    logger.info(f"[WRITE] Clean data written to {PipelineConfig.S3_PROCESSED}/orders/")

    # Rejected records: one flat folder per run date, for investigation
    n_rejected = rejected_df.count()
    if n_rejected > 0:
        (
            rejected_df
            .coalesce(1)
            .write
            .mode("overwrite")
            .parquet(f"{PipelineConfig.S3_REJECTED}/orders/dt={process_date}/")
        )
        logger.info(f"[WRITE] {n_rejected:,} rejected records written to dead-letter")

    # Metrics sidecar (read by the Airflow quality gate) — written with boto3.
    # MINIO_ENDPOINT is set locally; on AWS it is absent and boto3 uses real S3.
    metrics_key = f"metrics/orders/dt={process_date}/metrics.json"
    s3 = boto3.client(
        "s3",
        endpoint_url=os.environ.get("MINIO_ENDPOINT"),
        aws_access_key_id=os.environ.get("MINIO_ACCESS_KEY"),
        aws_secret_access_key=os.environ.get("MINIO_SECRET_KEY"),
    )
    s3.put_object(Bucket=PipelineConfig.S3_BUCKET, Key=metrics_key,
                  Body=json.dumps(metrics, indent=2))
    logger.info(f"[WRITE] Quality metrics written to s3://{PipelineConfig.S3_BUCKET}/{metrics_key}")
```

### Step 1.6: Main Entry Point

```python
# ── Main ────────────────────────────────────────────────────────
def main(process_date: str) -> None:
    logger.info("=" * 60)
    logger.info("URBANGEAR DAILY SALES ETL")
    logger.info(f"Processing date: {process_date}")
    logger.info(f"Started: {datetime.now().isoformat()}")
    logger.info("=" * 60)

    spark = create_spark_session("urbangear_daily_sales_etl")

    try:
        raw_df                = extract_raw_data(spark, process_date)
        clean_df, rejected_df = tag_and_route(raw_df, process_date)
        transformed_df        = transform_clean(clean_df, process_date)
        metrics               = collect_metrics(raw_df, transformed_df, rejected_df, process_date)
        write_outputs(transformed_df, rejected_df, metrics, process_date)

        logger.info("=" * 60)
        logger.info(f"ETL COMPLETE — {process_date}")
        logger.info(f"Clean: {metrics['clean_count']:,} | Rejected: {metrics['rejected_count']:,} "
                    f"| Rate: {metrics['reject_rate']:.2%}")
        logger.info("=" * 60)
    finally:
        spark.stop()


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    # Required on purpose: a batch job must be a pure function of its date.
    # Never default to "yesterday" — backfills and retries need explicit dates.
    parser.add_argument("--process_date", required=True, help="YYYY-MM-DD")
    # parse_known_args: AWS Glue adds its own arguments (e.g. --JOB_NAME); ignore them
    args, _ = parser.parse_known_args()
    main(args.process_date)
```

**Run it** (inside the Airflow container) and check the counts:

```powershell
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-01
```

Expected in the log: `[EXTRACT] 1,968 records read`, `[QUALITY] Clean: 1,909 | Rejected: 59`, and `ETL COMPLETE — 2026-07-01` with `Rate: 3.00%`. The run writes `processed/orders/order_year=2026/order_month=7/order_day=1/`, `rejected/orders/dt=2026-07-01/` and `metrics/orders/dt=2026-07-01/metrics.json`.

---

## PART 2: AIRFLOW DAG (90 min)

All of Part 2 is one file: `C:\DataEngineering\sk03\dags\urbangear_daily_sales_pipeline.py` (Granular Labs 05–07 build it step by step; Lab 08 adds the dbt task).

### Step 2.1: DAG Setup and Callbacks

**WHY:** The default_args block defines the fallback behavior for every task in the DAG. Centralize retry policy and alerting here rather than repeating on every operator.

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
```

> **On AWS (preview of SK-04):** alerts usually go to Slack through the Slack provider:
>
> ```python
> from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator
>
> def _slack_failure_alert(context):
>     ti = context["task_instance"]
>     SlackWebhookOperator(
>         task_id="slack_failure_alert",
>         slack_webhook_conn_id="slack_webhook",
>         message=f":red_circle: *UrbanGear Pipeline FAILED*\n*Task:* `{ti.task_id}`\n"
>                 f"*Date:* {context['ds']}\n*Log:* <{ti.log_url}|View Task Log>",
>     ).execute(context=context)
> ```
>
> Locally, `_send_alert` does the same job: it posts to Slack only if `SLACK_WEBHOOK_URL` is set (add it to the Airflow environment in `docker-compose.yml`) and always writes an `[ALERT]` line to the task log. `email_on_failure` needs an SMTP server, so it is off locally.

### Step 2.2: Python Callables

```python
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
```

### Step 2.3: DAG Definition and Task Dependencies

```python
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

Design notes:

- **Wait, then check.** The sensor waits for `part-*.json` (`wildcard_match=True` is required for a `*` pattern); `check_source_data` then counts the files and pushes the count to XCom.
- **Quality gate.** `quality_gate` returns `"load_to_warehouse.truncate_staging"` (≤ 5%) or `"alert_quality_failure"` (> 5%). The abort task alerts and then fails on purpose, so an aborted run is red.
- **Trigger rules.** `alert_success` keeps the default `all_success`, so it runs only after dbt succeeds; on the abort path it is skipped. `pipeline_end` joins both branches with `none_failed_min_one_success`.
- **Load.** `truncate_staging` deletes this date's rows, `copy_to_staging` inserts the date's processed Parquet (`jobs/load_to_warehouse.py`, Granular Lab 07). Both steps are keyed by the date, so the load is idempotent.

> **On AWS (preview of SK-04):** the Spark and load tasks become
>
> ```python
> from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
> from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
>
> run_spark_etl = GlueJobOperator(
>     task_id="run_spark_etl",
>     job_name="urbangear_daily_sales_etl",
>     script_args={"--process_date": "{{ ds }}"},
>     aws_conn_id="aws_default", region_name="us-east-1",
>     wait_for_completion=True, execution_timeout=timedelta(hours=3),
> )
>
> with TaskGroup(group_id="load_to_redshift") as load_to_redshift:
>     truncate_staging = SQLExecuteQueryOperator(
>         task_id="truncate_staging", conn_id="redshift_default",
>         sql="DELETE FROM staging.daily_orders WHERE etl_process_date = '{{ ds }}'::date;",
>     )
>     copy_to_staging = S3ToRedshiftOperator(
>         task_id="copy_to_staging", schema="staging", table="daily_orders",
>         s3_bucket="urbangear-data-lake",
>         s3_key="processed/orders/order_year={{ macros.ds_format(ds, '%Y-%m-%d', '%Y') }}/"
>                "order_month={{ macros.ds_format(ds, '%Y-%m-%d', '%-m') }}/"
>                "order_day={{ macros.ds_format(ds, '%Y-%m-%d', '%-d') }}/",
>         redshift_conn_id="redshift_default", aws_conn_id="aws_default",
>         copy_options=["FORMAT AS PARQUET"],
>         method="APPEND",   # the DELETE above already removed this date; "REPLACE" would wipe EVERY date
>     )
>     truncate_staging >> copy_to_staging
> ```
>
> and the branch target becomes `"load_to_redshift.truncate_staging"`. Note that Parquet `COPY` maps columns by position and does not load the partition folders' values, so in production the staging table's column list must match the Parquet files exactly.

### Step 2.4: Run It

```powershell
docker exec -it airflow-scheduler airflow dags list-import-errors
docker exec -it airflow-scheduler airflow dags unpause urbangear_daily_sales_pipeline
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-03
docker exec -it airflow-scheduler airflow dags trigger urbangear_daily_sales_pipeline -e 2026-07-05
```

(Do Part 3 first if you want `run_dbt_transformations` to succeed on the good day.) Expected: the 2026-07-03 run passes the gate (`reject_rate=2.69%`) and loads 1,950 rows; the 2026-07-05 run stops at the gate (`reject_rate=11.78% > 5%`), `alert_quality_failure` is red, the load and dbt tasks are skipped. To load history use `airflow dags backfill urbangear_daily_sales_pipeline -s 2026-07-01 -e 2026-07-04T04:00:00` (runs are at 04:00, hence the time on the end date). Pause the DAG when done.

---

## PART 3: dbt TRANSFORMATION LAYER (60 min)

The dbt project lives in `C:\DataEngineering\sk03\dbt\urbangear` (`/opt/airflow/dbt/urbangear` in the container); the profile in `dbt\profiles\profiles.yml`. Run dbt commands inside the container (Granular Lab 08):

```powershell
docker exec -it airflow-scheduler bash
```

```bash
cd /opt/airflow/dbt && dbt init urbangear --skip-profile-setup && rm -rf urbangear/models/example
export DBT_PROFILES_DIR=/opt/airflow/dbt/profiles && cd urbangear
```

### Step 3.0: Project Setup

`dbt_project.yml` (replace the generated one):

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

`../profiles/profiles.yml` — credentials come from the container's environment:

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

`packages.yml` — the tests in Step 3.3 use `dbt_utils` (run `dbt deps` after creating it):

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
```

`macros/generate_schema_name.sql` — makes `schema='staging'` / `schema='analytics'` mean exactly those schemas:

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

`models/staging/sources.yml` — declares the `source('staging', 'daily_orders')` used below, with a freshness check (`dbt source freshness`):

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

### Step 3.1: Staging Model

**WHY:** The staging layer is a 1:1 mirror of the source with types cleaned and names standardized. No business logic lives here. This keeps the mart models readable and makes source schema changes easy to isolate. Money columns are cast to `NUMERIC` so the mart can round them (Postgres has no `ROUND(double precision, int)`).

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

### Step 3.2: Mart Model

**Grain:** one row per order **line** (an order can contain several products), so the unique key is `order_id` + `product_id`.

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

### Step 3.3: Schema Tests

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

Plus a singular test that uses the `process_date` variable the DAG passes with `--vars`:

```sql
-- tests/assert_process_date_loaded.sql
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

Run them:

```bash
dbt deps
dbt source freshness
dbt run  --select tag:daily
dbt test --select tag:daily --vars '{process_date: 2026-07-03}'
```

---

## PART 4: VERIFICATION (20 min)

Run in the warehouse (`docker exec -it warehouse psql -U dwh_user -d warehouse`). On a live daily pipeline you'd filter on `CURRENT_DATE - 1`; with this lab's fixed sample dates, use the date you ran.

```sql
-- Run these checks after the pipeline completes to verify correctness.

-- 1. Record count for the run
SELECT
    order_day,
    COUNT(*)                      AS line_item_count,
    COUNT(DISTINCT order_id)      AS order_count,
    COUNT(DISTINCT customer_id)   AS unique_customers,
    ROUND(SUM(net_revenue), 2)    AS total_net_revenue
FROM analytics.fct_daily_sales
WHERE order_day = DATE '2026-07-03'      -- production: CURRENT_DATE - 1
GROUP BY order_day;

-- 2. Platform distribution (should be stable day-over-day)
SELECT
    platform,
    COUNT(*) AS order_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct
FROM analytics.fct_daily_sales
WHERE order_day = DATE '2026-07-03'
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

-- 5. Verify nothing from the rejected bad day reached the warehouse
SELECT COUNT(*) AS bad_day_rows
FROM analytics.fct_daily_sales
WHERE order_day = DATE '2026-07-05';
-- Expected: 0

-- 6. Verify idempotency: note query #1's count, re-run the pipeline for the same date
--    (or `dbt run --select tag:daily --full-refresh`), and run query #1 again.
--    The count must be identical.
```

For 2026-07-03, query 1 should return 1,950 line items.

---

## REFLECTION QUESTIONS

1. The quality gate branches to `load_to_warehouse.truncate_staging` (a task inside a TaskGroup) or `alert_quality_failure`. What happens to `run_dbt_transformations`, `alert_success` and `pipeline_end` when the `alert_quality_failure` branch is taken? Walk through the trigger_rule logic, and explain why `pipeline_end` needs `none_failed_min_one_success`.

2. The Spark job uses `mode=PERMISSIVE` and writes corrupted records to a dead-letter path. Under what circumstance would you change this to `mode=FAILFAST`, and what would you need to add to the Airflow DAG to handle a FAILFAST failure gracefully?

3. The dbt staging model uses a `-3 day` buffer on the incremental watermark (`MAX(etl_processed_at) - INTERVAL '3 days'`). Why is this buffer necessary? What could go wrong without it?

4. On AWS, `run_spark_etl` is a `GlueJobOperator` that inherits `retries=3` from `default_args`, and the Glue job definition itself can also have its own "Maximum retries" setting (say 2). If the Glue job keeps failing, how many total Spark attempts are made? Draw the retry sequence, and say which setting you would keep.

5. You need to add a new source (a 4th sales platform) that uploads its data as CSV instead of JSON. Walk through every file you would need to change: the PySpark job, the Airflow DAG, the dbt models, and the schema.yml tests.
