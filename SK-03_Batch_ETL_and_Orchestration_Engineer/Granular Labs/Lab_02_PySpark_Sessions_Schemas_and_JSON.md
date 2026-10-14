# Lab 02 — PySpark Sessions, Schemas and JSON

> **Goal:** Start the guided lab's `etl_job.py` — `PipelineConfig`, the explicit `RAW_ORDER_SCHEMA`, and the tuned `create_spark_session()` — and use it to read the raw JSON orders, including the corrupt lines.
> **Time:** 180–240 min | **Difficulty:** Beginner | **Prerequisites:** Lab 01
> **Guided lab coverage:** Part 1 Step 1.1 (imports, logging, `PipelineConfig`) and Step 1.2 (`RAW_ORDER_SCHEMA`, `TimestampType`, `create_spark_session` with AQE, shuffle partitions, dynamic partition overwrite, Kryo); PERMISSIVE mode and `_corrupt_record` from Step 1.3

## Learning objectives

By the end of this lab you can:

- Explain Spark's driver/executor model, lazy **transformations** vs **actions**, and what a **shuffle** is.
- Explain every setting in the guided lab's `create_spark_session()`.
- Explain why production jobs declare a schema (`StructType`) instead of inferring it.
- Read JSON Lines in **PERMISSIVE** mode and find corrupt lines through `_corrupt_record`.
- Read a physical plan (`explain()`) and find the job's stages in the Spark UI.

## Steps

### Step 1 — Know the moving parts

- **Driver** — the process running *your* Python script; it builds the plan and hands out work.
- **Executors** — worker processes that hold chunks of data (**partitions**) and run **tasks** on them in parallel.
- **DataFrame** — a distributed table with named, typed columns.
- **Transformations** (`select`, `filter`, `withColumn`, `groupBy`) only *describe* work — they return instantly.
- **Actions** (`count`, `show`, `collect`, `write`) *run* the plan. Only now does Spark read data.
- A **shuffle** moves rows between executors so equal keys meet (`groupBy`, `join`, window functions). Shuffles split a job into **stages** and are where most slowness lives.

In this module the pipeline runs Spark in **local mode** inside the Airflow container (driver and executors in one process). On AWS, the guided lab's Glue job runs the *same code* on a managed cluster.

**Expected output:** none.

✅ **Checkpoint:** you can explain "transformation vs action" in one sentence each.

### Step 2 — Start `etl_job.py`: imports, logging and `PipelineConfig`

Create `C:\DataEngineering\sk03\jobs\etl_job.py` with this first part (it is the top of the guided lab's Step 1.1, adapted to the local stack). Some imports are used only in Labs 03–04 — keep them; the file grows into the complete job.

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

- **`logging` instead of `print`** — every line gets a timestamp and a level (`INFO`, `ERROR`). Airflow captures these lines into its task logs, where `[EXTRACT]`, `[QUALITY]` tags make them easy to search.
- **`PipelineConfig`** — every path, threshold and business rule in one class. Moving from MinIO to AWS is a config change, not a code change.
- **`s3a://`** — the S3 connector Spark uses (the guided lab writes `s3://`, which is what AWS Glue accepts; with open-source Spark you write `s3a://`).
- **`MIN_RECORD_COUNT = 1_000`** and **`SHUFFLE_PARTITIONS = 8`** — sized for ~2,000 lines/day on a laptop. The guided lab's production values (100,000 and 200) are in the comments.

**Expected output:** none yet.

✅ **Checkpoint:** `jobs\etl_job.py` exists and ends with the `SHUFFLE_PARTITIONS` line.

### Step 3 — Add the schema and the SparkSession factory

Append this to `etl_job.py` (guided lab Step 1.2):

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

What each piece does:

| Setting | Why |
|---|---|
| `StructType([...])` with `TimestampType` for `order_date` | The schema is your **contract** with the source. Spark parses `2026-07-01T20:05:41` straight into a timestamp; no guessing. |
| `_corrupt_record` field | In PERMISSIVE mode Spark stores the raw text of any unparseable line here. **It must be declared** in an explicit schema, otherwise Spark drops it and corrupt lines silently become all-NULL rows. |
| `spark.sql.adaptive.enabled` (AQE) | Spark re-optimises *while running*, using real statistics: merges tiny shuffle partitions, splits skewed ones. |
| `spark.sql.adaptive.coalescePartitions.enabled` | The AQE feature that merges small shuffle partitions. |
| `spark.sql.shuffle.partitions` | How many partitions a shuffle creates (Spark's default is 200 — far too many for 2,000 rows). |
| `spark.sql.sources.partitionOverwriteMode=dynamic` | `mode("overwrite")` replaces **only the partitions you write** (e.g. one day), not the whole table. Lab 04 relies on this. |
| `spark.sql.parquet.compression.codec=snappy` | Fast compression for Parquet output. |
| `spark.serializer=KryoSerializer` | Faster, smaller serialization when Spark ships objects between JVMs. |
| `spark.sql.session.timeZone=UTC` | Timestamps mean the same thing on every machine. |
| `fs.s3a.*` settings (only when `MINIO_ENDPOINT` is set) | Point Spark's S3 connector at MinIO. On AWS these env vars don't exist, so the block is skipped. |

**Expected output:** none yet.

✅ **Checkpoint:** `etl_job.py` now ends with `return builder.getOrCreate()`.

### Step 4 — Verify the session and its settings

```powershell
docker exec -it airflow-scheduler python -c "import sys; sys.path.insert(0,'/opt/airflow/jobs'); from etl_job import create_spark_session; s=create_spark_session('verify'); print('SPARK', s.version, s.conf.get('spark.sql.adaptive.enabled'), s.conf.get('spark.sql.shuffle.partitions'), s.conf.get('spark.sql.sources.partitionOverwriteMode')); s.stop()"
```

**Expected output** (after some `WARN` lines you can ignore):

```
SPARK 3.5.1 true 8 dynamic
```

✅ **Checkpoint:** you see `SPARK 3.5.1 true 8 dynamic`.

### Step 5 — See why inferred schemas are dangerous

Open an interactive Python session in the container and keep it open for Steps 5–8:

```powershell
docker exec -it airflow-scheduler python
```

```python
import sys; sys.path.insert(0, "/opt/airflow/jobs")
from pyspark.sql import functions as F
from etl_job import PipelineConfig, RAW_ORDER_SCHEMA, create_spark_session
spark = create_spark_session("lab02-explore")
path = f"{PipelineConfig.S3_RAW}/orders/dt=2026-07-01/"

inferred = spark.read.json(path)          # Spark scans the data and GUESSES types
inferred.printSchema()
```

**Expected output:**

```
root
 |-- _corrupt_record: string (nullable = true)
 |-- category: string (nullable = true)
 |-- customer_id: string (nullable = true)
 |-- discount_pct: double (nullable = true)
 |-- is_returned: boolean (nullable = true)
 |-- order_date: string (nullable = true)
 |-- order_id: string (nullable = true)
 |-- payment_method: string (nullable = true)
 |-- platform: string (nullable = true)
 |-- product_id: string (nullable = true)
 |-- product_name: string (nullable = true)
 |-- quantity: long (nullable = true)
 |-- shipping_city: string (nullable = true)
 |-- shipping_cost: double (nullable = true)
 |-- shipping_country: string (nullable = true)
 |-- shipping_state: string (nullable = true)
 |-- tax_amount: double (nullable = true)
 |-- unit_price: double (nullable = true)
```

Three problems: columns are alphabetical instead of the source order, `order_date` is a **string** (not a timestamp), and `quantity` is `long`. Inference also reads the whole file once just to guess, and its guesses change with the day's data. Production jobs never infer.

✅ **Checkpoint:** you can name two things inference got wrong.

### Step 6 — Read with the explicit schema (PERMISSIVE mode)

Still in the same Python session:

```python
df = (spark.read
      .schema(RAW_ORDER_SCHEMA)
      .option("mode", "PERMISSIVE")
      .option("columnNameOfCorruptRecord", "_corrupt_record")
      .json(path))                         # instant: nothing has been read yet (lazy)
df.printSchema()
print("lines:", df.count())                # ACTION: now Spark reads the file
df.select("order_id", "platform", "order_date", "product_id", "quantity").show(3, truncate=False)
```

**Expected output:**

```
root
 |-- order_id: string (nullable = true)
 |-- customer_id: string (nullable = true)
 |-- platform: string (nullable = true)
 |-- order_date: timestamp (nullable = true)
 |-- product_id: string (nullable = true)
 |-- product_name: string (nullable = true)
 |-- category: string (nullable = true)
 |-- quantity: integer (nullable = true)
 |-- unit_price: double (nullable = true)
 |-- discount_pct: double (nullable = true)
 |-- shipping_cost: double (nullable = true)
 |-- tax_amount: double (nullable = true)
 |-- payment_method: string (nullable = true)
 |-- shipping_country: string (nullable = true)
 |-- shipping_state: string (nullable = true)
 |-- shipping_city: string (nullable = true)
 |-- is_returned: boolean (nullable = true)
 |-- _corrupt_record: string (nullable = true)

lines: 1968
+------------------+---------+-------------------+----------+--------+
|order_id          |platform |order_date         |product_id|quantity|
+------------------+---------+-------------------+----------+--------+
|ORD-20260701-00000| Website |2026-07-01 20:05:41|P100      |2       |
|ORD-20260701-00000| Website |2026-07-01 20:05:41|P600      |1       |
|ORD-20260701-00000| Website |2026-07-01 20:05:41|P200      |3       |
+------------------+---------+-------------------+----------+--------+
```

- Column order and types now match the contract; `order_date` is a real `timestamp`.
- `order_id` shows `nullable = true` even though the schema says `False`: for files, Spark always treats columns as nullable (a file can't promise anything). Your **quality checks** (Lab 03) enforce non-null, not the schema.
- PERMISSIVE kept all 1,968 lines. The alternatives are `DROPMALFORMED` (silently deletes bad lines — never acceptable) and `FAILFAST` (one bad line kills the whole job).

✅ **Checkpoint:** `lines: 1968` and `order_date` is a timestamp.

### Step 7 — Find the corrupt lines (and meet a Spark rule)

```python
df.filter(F.col("_corrupt_record").isNotNull()).count()
```

**Expected output:** an error. The message begins approximately like this:

```
pyspark.errors.exceptions.captured.AnalysisException: [UNSUPPORTED_FEATURE.QUERY_ONLY_CORRUPT_RECORD_COLUMN] ...
Queries from raw JSON/CSV files are disallowed when the referenced columns only include the internal corrupt record column ...
```

Spark refuses a query that reads *only* `_corrupt_record` straight from the file, because it would skip parsing the other columns and get the answer wrong. The fix — and the reason the guided lab's job uses `.cache()` in Lab 03 — is to cache the parsed data first:

```python
df = df.cache()
bad = df.filter(F.col("_corrupt_record").isNotNull())
print("corrupt:", bad.count())
bad.select("order_id", "_corrupt_record").show(2, truncate=60)
```

**Expected output:**

```
corrupt: 15
+--------+------------------------------------------------------------+
|order_id|                                             _corrupt_record|
+--------+------------------------------------------------------------+
|    NULL|{"order_id": "ORD-20260701-00123", "customer_id": "CUST-0...|
|    NULL|{"order_id": "ORD-20260701-00147", "customer_id": "CUST-0...|
+--------+------------------------------------------------------------+
```

Every other column is `NULL` for a corrupt line; the original text survives in `_corrupt_record`, ready for the dead-letter zone. **15 matches the count you found with plain Python in Lab 01, Step 6.**

✅ **Checkpoint:** `corrupt: 15`.

### Step 8 — Lazy evaluation, a shuffle, and `explain()`

```python
by_platform = df.groupBy("platform").count().orderBy(F.desc("count"))   # instant
by_platform.show()                                                       # action
by_platform.explain()
```

**Expected output** (row order of equal counts may vary; plan abbreviated):

```
+-------------+-----+
|     platform|count|
+-------------+-----+
|      website|  640|
|  marketplace|  555|
|   mobile_app|  538|
|     Website |   80|
| Marketplace |   63|
|  Mobile_App |   62|
|          fax|   15|
|         NULL|   15|
+-------------+-----+

== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- Sort [count#... DESC NULLS LAST], true, 0
   +- Exchange rangepartitioning(count#... DESC NULLS LAST, 8), ...
      +- HashAggregate(keys=[platform#...], functions=[count(1)])
         +- Exchange hashpartitioning(platform#..., 8), ENSURE_REQUIREMENTS, ...
            +- HashAggregate(keys=[platform#...], functions=[partial_count(1)])
               +- InMemoryTableScan [platform#...]
```

- The raw `platform` column is messy: `" Website "` and `website` are the same channel. Lab 03 normalises with `lower(trim(...))` *before* validating; `fax` and `NULL` are genuinely invalid.
- **`Exchange` = shuffle.** `hashpartitioning(platform, 8)` is the `groupBy`; the `8` is your `SHUFFLE_PARTITIONS`. `AdaptiveSparkPlan` means AQE is on.
- `InMemoryTableScan` — Spark read from the cache, not MinIO.

Type `spark.stop()` then `exit()`.

✅ **Checkpoint:** you found the two `Exchange` operators in the plan.

### Step 9 — Watch a job in the Spark UI (cluster mode)

The local-mode session's UI is hidden inside the Airflow container, so submit a small job to the real Spark cluster from Lab 00, whose UI is published on http://localhost:8081.

Create `C:\DataEngineering\sk03\jobs\lab02_agg.py`:

```python
"""Lab 02: revenue by category over all raw days, run on the Spark cluster."""
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("lab02-agg").getOrCreate()   # settings come from spark-submit
raw = spark.read.json("s3a://urbangear-data-lake/raw/orders/")    # all dt= folders at once
(raw.groupBy("category")
    .agg(F.round(F.sum(F.col("quantity") * F.col("unit_price")), 2).alias("revenue"),
         F.count("*").alias("lines"))
    .orderBy(F.desc("revenue"))
    .show())
input("Open http://localhost:8081 -> Running Applications -> lab02-agg -> Application Detail UI. Press Enter to finish.")
spark.stop()
```

Submit it from the `spark-master` container (`jobs\` is mounted there at `/opt/jobs`). This container has no MinIO env vars, so the S3A settings are passed explicitly, and `--packages` downloads the S3A JARs (cached in `/tmp/.ivy2`):

```powershell
docker exec -it spark-master spark-submit `
  --master spark://spark-master:7077 `
  --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 `
  --conf spark.jars.ivy=/tmp/.ivy2 `
  --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000 `
  --conf spark.hadoop.fs.s3a.access.key=minioadmin `
  --conf spark.hadoop.fs.s3a.secret.key=minioadmin123 `
  --conf spark.hadoop.fs.s3a.path.style.access=true `
  /opt/jobs/lab02_agg.py
```

**Expected output** (after the JAR download on first run; the last decimal can differ slightly because floating-point sums depend on order):

```
+---------+---------+-----+
| category|  revenue|lines|
+---------+---------+-----+
|  Shelter| 835402.0| 1690|
| Sleeping| 509034.0| 1620|
|     Bags| 361759.8| 1635|
|  Cooking| 169032.5| 1553|
|Hydration|149551.05| 1708|
| Lighting| 82538.71| 1660|
|     NULL|     NULL|   99|
+---------+---------+-----+
```

(The 99 `NULL` lines are the corrupt lines across all five days — inference created `_corrupt_record` for them. Revenue here still includes the dirty rows; Lab 03 removes them.)

While the script waits at `input(...)`, open http://localhost:8081 → **Running Applications** → `lab02-agg` → **Application Detail UI**:

1. **Jobs** — one job per action.
2. **Stages** — the `groupBy` job has stages split at the shuffle; note "Shuffle Read/Write".
3. **Executors** — your one worker and its cores.

Press Enter in PowerShell to finish.

✅ **Checkpoint:** you saw the stages of `lab02-agg` in the Spark UI.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'etl_job'` | Path not added, or file not in `jobs\` | Run `sys.path.insert(0, "/opt/airflow/jobs")` first; check the file location |
| `Path does not exist: s3a://urbangear-data-lake/raw/orders/dt=2026-07-01` | Lab 01 data missing | Re-run Lab 01 Step 5 |
| `ClassNotFoundException: org.apache.hadoop.fs.s3a.S3AFileSystem` | Image built without the S3A JARs | Check the Dockerfile's `curl ... hadoop-aws` lines; `docker compose up -d --build` |
| `basedir must be absolute: ?/.ivy2/local` (Step 9) | The Spark container user has no home folder | Keep `--conf spark.jars.ivy=/tmp/.ivy2` |
| `NoSuchBucket` / `403` from s3a | Wrong endpoint or credentials | Endpoint `http://minio:9000`, keys `minioadmin`/`minioadmin123`, `path.style.access=true` |
| Step 9 UI link shows nothing | The app already finished | That is why the script waits at `input()`; re-run it |
| `corrupt: 0` | `_corrupt_record` missing from the schema | Copy `RAW_ORDER_SCHEMA` exactly — the last field is `_corrupt_record` |

## Key takeaways

- `etl_job.py` now has the guided lab's Steps 1.1–1.2: logging, `PipelineConfig`, `RAW_ORDER_SCHEMA`, `create_spark_session()`.
- Never infer schemas in production: declare a `StructType`, including `_corrupt_record` when reading in PERMISSIVE mode.
- Transformations are lazy; actions run the plan. `Exchange` in `explain()` is a shuffle.
- AQE, sensible shuffle partitions, Kryo and dynamic partition overwrite are set once, in the session factory.
- Queries that touch only `_corrupt_record` need the DataFrame cached first.

## Next

[Lab 03 — Data Quality Flags and Routing](Lab_03_Data_Quality_Flags_and_Routing.md): add `extract_raw_data()` (with the volume check) and `tag_and_route()` to split clean rows from rejected ones, with a reason for every rejection.
