# Lab 04 — Transformations, Window Functions and Partitioned Writes

> **Goal:** Finish `etl_job.py` exactly as the guided lab defines it — business transformations with window functions, quality metrics, partitioned Parquet with dynamic overwrite, a dead-letter output, a metrics JSON sidecar, and a `--process_date` entry point — then run it with `spark-submit`.
> **Time:** 180–240 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 03
> **Guided lab coverage:** Part 1 Step 1.4 (`transform_clean`: revenue measures, text standardisation, date parts, `Window` aggregates, ETL metadata), Step 1.5 (`collect_metrics`, `write_outputs`: `partitionBy` + dynamic overwrite, dead-letter, metrics sidecar) and Step 1.6 (`main`, `argparse`)

## Learning objectives

By the end of this lab you can:

- Add calculated columns with `withColumn`, `round` and `coalesce`.
- Use a **window function** to attach order-level totals to every line, without a `groupBy`.
- Write **partitioned Parquet** and explain how **dynamic partition overwrite** makes re-runs safe.
- Produce a machine-readable **metrics sidecar** that an orchestrator can act on.
- Run a PySpark job with `spark-submit` and a required `--process_date` argument.

## Steps

### Step 1 — Add `transform_clean()`

Append to `C:\DataEngineering\sk03\jobs\etl_job.py` (guided lab Step 1.4):

```python
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

- **Revenue measures:** `gross = quantity × unit_price`, `discount = gross × discount_pct / 100`, `net = gross − discount`, `total = net + shipping + tax`. `coalesce(discount_pct, 0)` treats a missing discount as "no discount" instead of turning the whole result into NULL.
- **Standardisation:** `platform` lower-case and trimmed, `payment_method` and `shipping_country` upper-case — so `" Website "` and `website` group together downstream.
- **Date parts:** `order_year`, `order_month`, `order_day` become the folder structure of the output; `day_of_week` uses Spark's convention **1 = Sunday … 7 = Saturday**, so `is_weekend` is `isin(1, 7)`.
- **Window functions:** `Window.partitionBy("order_id")` defines "all lines of the same order". `F.count("*").over(w)` and `F.sum("net_revenue").over(w)` put the order's item count and total on **every** line — a `groupBy` would collapse the lines instead.
- **ETL metadata:** `etl_processed_at` (when) and `etl_process_date` (which run) let you trace any warehouse row back to the run that produced it. dbt's incremental models use `etl_processed_at` in Lab 08.
- The internal flag columns and `rejection_reason` are dropped — clean rows don't need them.

**Expected output:** none yet.

✅ **Checkpoint:** `etl_job.py` contains `def transform_clean`.

### Step 2 — See the window function at work

```powershell
docker exec -it airflow-scheduler python
```

```python
import sys; sys.path.insert(0, "/opt/airflow/jobs")
from pyspark.sql import functions as F
from etl_job import create_spark_session, extract_raw_data, tag_and_route, transform_clean
spark = create_spark_session("lab04")

raw = extract_raw_data(spark, "2026-07-01")
clean, rejected = tag_and_route(raw, "2026-07-01")
out = transform_clean(clean, "2026-07-01")

(out.filter(F.col("order_id") == "ORD-20260701-00000")
    .select("product_id", "quantity", "gross_revenue", "discount_amount", "net_revenue",
            "total_amount", "order_item_count", "order_total_revenue", "platform", "payment_method")
    .orderBy("product_id")
    .show())
out.groupBy("order_item_count").count().orderBy("order_item_count").show()
print("high-value lines:", out.filter("is_high_value").count())
spark.stop()
```

**Expected output** (log lines omitted):

```
+----------+--------+-------------+---------------+-----------+------------+----------------+-------------------+--------+--------------+
|product_id|quantity|gross_revenue|discount_amount|net_revenue|total_amount|order_item_count|order_total_revenue|platform|payment_method|
+----------+--------+-------------+---------------+-----------+------------+----------------+-------------------+--------+--------------+
|      P100|       2|       179.98|           18.0|     161.98|      188.03|               3|             314.54| website|     GIFT_CARD|
|      P200|       3|        136.5|          13.65|     122.85|       146.5|               3|             314.54| website|     GIFT_CARD|
|      P600|       1|        34.95|           5.24|      29.71|       38.76|               3|             314.54| website|     GIFT_CARD|
+----------+--------+-------------+---------------+-----------+------------+----------------+-------------------+--------+--------------+

+----------------+-----+
|order_item_count|count|
+----------------+-----+
|               1|  344|
|               2|  722|
|               3|  843|
+----------------+-----+

high-value lines: 0
```

- All three lines of the order carry the same `order_item_count` (3) and `order_total_revenue` (314.54) — that is a window function.
- In the second table, `count` is **lines**: 344 one-line orders, 361 two-line orders (722 lines), 281 three-line orders (843 lines) = 1,909 clean lines.
- `" Website "` became `website`; `gift_card` became `GIFT_CARD`.
- No order is near the $50,000 `MAX_ORDER_VALUE`, so `is_high_value` is false everywhere — on real data the flag picks out orders for review.

✅ **Checkpoint:** `order_total_revenue` is `314.54` on all three lines.

### Step 3 — Add `collect_metrics()` and `write_outputs()`

Append to `etl_job.py` (guided lab Step 1.5):

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

- **`partitionBy("order_year", "order_month", "order_day")`** writes folders like `processed/orders/order_year=2026/order_month=7/order_day=1/`. Readers filtering on those columns open only the matching folders (**partition pruning**).
- **`mode("overwrite")` + `partitionOverwriteMode=dynamic`** (set in Lab 02) replaces *only* the partitions present in this DataFrame — one day. Without dynamic mode, `overwrite` would delete **every other day** in `processed/orders/`. That is the most famous foot-gun in Spark batch jobs.
- **`.repartition(order_year, order_month, order_day)`** before writing puts each day's rows in one task, so each partition folder gets one file instead of many tiny ones.
- **Rejected rows** go to `rejected/orders/dt=<date>/` with all their flag columns and `_corrupt_record`, so anyone can see *why* each row failed. `coalesce(1)` = one small file.
- **The metrics sidecar** `metrics/orders/dt=<date>/metrics.json` is tiny JSON the Airflow quality gate reads in Lab 07. It is written with boto3; `endpoint_url=os.environ.get("MINIO_ENDPOINT")` means "MinIO locally, real S3 when the variable isn't set".
- **`rule_counts`** groups the rejected rows by `rejection_reason` — one count per rule.

**Expected output:** none yet.

✅ **Checkpoint:** `etl_job.py` contains `def write_outputs`.

### Step 4 — Add `main()` and the command-line entry point

Append to `etl_job.py` (guided lab Step 1.6):

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

- **`--process_date` is required.** The job never guesses "yesterday" from the clock. A job that computes its own date can only ever process one day, so backfills and retries would load the wrong data. Airflow passes the date explicitly (`{{ ds }}`, Lab 05).
- **`try / finally: spark.stop()`** releases memory even when a step raises.

Check the file is complete and valid Python:

```powershell
docker exec -it airflow-scheduler python -m py_compile /opt/airflow/jobs/etl_job.py
docker exec -it airflow-scheduler python /opt/airflow/jobs/etl_job.py
```

**Expected output:** the first command prints nothing (valid syntax). The second prints

```
usage: etl_job.py [-h] --process_date PROCESS_DATE
etl_job.py: error: the following arguments are required: --process_date
```

✅ **Checkpoint:** `py_compile` is silent and the job refuses to run without a date.

### Step 5 — Run the complete job with `spark-submit`

`spark-submit` is the standard way to launch a Spark application (on Glue, EMR, Kubernetes or a laptop). `--master 'local[2]'` = local mode with 2 cores — exactly what the Airflow task will run in Lab 06.

```powershell
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-01
```

**Expected output** (only the job's own `INFO` lines shown; timestamps and the long Spark log differ):

```
[INFO] ============================================================
[INFO] URBANGEAR DAILY SALES ETL
[INFO] Processing date: 2026-07-01
[INFO] Started: 2026-...
[INFO] ============================================================
[INFO] Reading raw data from: s3a://urbangear-data-lake/raw/orders/dt=2026-07-01/
[INFO] [EXTRACT] 1,968 records read
[INFO] [QUALITY] Clean: 1,909 | Rejected: 59
[INFO] [METRICS] {'process_date': '2026-07-01', 'total_count': 1968, 'clean_count': 1909, 'rejected_count': 59, 'reject_rate': 0.03, 'rule_counts': {'corrupt_json': 15, 'future_order_date': 15, 'invalid_platform': 15, 'negative_quantity': 2, 'null_critical_field': 12}, 'generated_at': '2026-...'}
[INFO] [WRITE] Clean data written to s3a://urbangear-data-lake/processed/orders/
[INFO] [WRITE] 59 rejected records written to dead-letter
[INFO] [WRITE] Quality metrics written to s3://urbangear-data-lake/metrics/orders/dt=2026-07-01/metrics.json
[INFO] ============================================================
[INFO] ETL COMPLETE — 2026-07-01
[INFO] Clean: 1,909 | Rejected: 59 | Rate: 3.00%
[INFO] ============================================================
```

✅ **Checkpoint:** `ETL COMPLETE — 2026-07-01` with `Clean: 1,909 | Rejected: 59 | Rate: 3.00%`.

### Step 6 — Inspect the three outputs

In the MinIO console (http://localhost:9001) open `urbangear-data-lake` and find:

- `processed/orders/order_year=2026/order_month=7/order_day=1/part-00000-....snappy.parquet`
- `rejected/orders/dt=2026-07-01/part-00000-....snappy.parquet`
- `metrics/orders/dt=2026-07-01/metrics.json` — click it → **Preview**.

**Expected output** (`metrics.json`; `generated_at` differs):

```json
{
  "process_date": "2026-07-01",
  "total_count": 1968,
  "clean_count": 1909,
  "rejected_count": 59,
  "reject_rate": 0.03,
  "rule_counts": {
    "corrupt_json": 15,
    "future_order_date": 15,
    "invalid_platform": 15,
    "negative_quantity": 2,
    "null_critical_field": 12
  },
  "generated_at": "2026-..."
}
```

(Spark also writes `_SUCCESS` marker files next to the Parquet — a signal that the write completed.)

✅ **Checkpoint:** all three outputs exist and `metrics.json` shows `"reject_rate": 0.03`.

### Step 7 — Prove dynamic overwrite and idempotency

Process the other normal days, then run 2026-07-01 **again**:

```powershell
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-02
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-03
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-04
docker exec -it airflow-scheduler spark-submit --master "local[2]" /opt/airflow/jobs/etl_job.py --process_date 2026-07-01
```

Now count what is in `processed/`:

```powershell
docker exec -it airflow-scheduler python -c "import sys; sys.path.insert(0,'/opt/airflow/jobs'); from etl_job import create_spark_session; s=create_spark_session('check'); df=s.read.parquet('s3a://urbangear-data-lake/processed/orders/'); df.groupBy('order_year','order_month','order_day','is_weekend').count().orderBy('order_day').show(); s.stop()"
```

**Expected output:**

```
+----------+-----------+---------+----------+-----+
|order_year|order_month|order_day|is_weekend|count|
+----------+-----------+---------+----------+-----+
|      2026|          7|        1|     false| 1909|
|      2026|          7|        2|     false| 1924|
|      2026|          7|        3|     false| 1950|
|      2026|          7|        4|      true| 1933|
+----------+-----------+---------+----------+-----+
```

- **Day 1 still has 1,909 rows** after two runs: the re-run *replaced* its partition (idempotent), and days 2–4 were untouched (dynamic overwrite).
- `order_year/order_month/order_day` came back as columns — Spark reads them from the folder names.
- 2026-07-04 is a Saturday, so `is_weekend` is `true`.

Do **not** process 2026-07-05 yet — in Lab 07 the pipeline does that, and the quality gate stops it.

✅ **Checkpoint:** four partitions with 1,909 / 1,924 / 1,950 / 1,933 rows.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| After a re-run, other days disappeared from `processed/` | Static overwrite | `create_spark_session` must set `spark.sql.sources.partitionOverwriteMode=dynamic` |
| Day 1 shows 3,818 rows | Wrote with `mode("append")` | Use `mode("overwrite")` with dynamic mode |
| `spark-submit: command not found` | Running on Windows, or old image | Prefix with `docker exec -it airflow-scheduler`; rebuild the image (Lab 00 Step 7) |
| `NoCredentialsError` when writing metrics | Env vars missing | Run inside `airflow-scheduler`; check `MINIO_*` in `docker-compose.yml` |
| `ValueError: Record count ... below the minimum threshold` | No/too little input for that date | Check `raw/orders/dt=<date>/` exists (Lab 01 Step 5) |
| `order_total_revenue` differs per line of one order | `Window` partitioned by the wrong column | `Window.partitionBy("order_id")` |
| `is_weekend` true on weekdays | Assumed 1 = Monday | Spark's `dayofweek`: 1 = Sunday, 7 = Saturday |

## Key takeaways

- `etl_job.py` is now the complete guided-lab job (Steps 1.1–1.6), runnable with `spark-submit ... --process_date <date>`.
- Window functions add group-level values to every row without collapsing rows.
- Partitioned Parquet + dynamic partition overwrite = re-run any day safely, without touching the others.
- Three outputs per run: clean (`processed/`), dead-letter (`rejected/`), and a metrics sidecar (`metrics/`) for the orchestrator.
- A batch job is a pure function of its date: the date is a required argument, never "today".

## Next

[Lab 05 — Airflow Fundamentals: Your First DAG](Lab_05_Airflow_Fundamentals_First_DAG.md): learn how Airflow schedules work, passes the date (`{{ ds }}`), retries, and alerts — before it runs this job every night.
