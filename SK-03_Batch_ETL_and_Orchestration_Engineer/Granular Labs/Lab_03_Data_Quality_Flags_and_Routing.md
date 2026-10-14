# Lab 03 — Data Quality Flags and Routing

> **Goal:** Add the guided lab's `extract_raw_data()` (with its volume check) and `tag_and_route()` to `etl_job.py`, so every raw line ends up either clean or rejected — with a recorded reason.
> **Time:** 150–210 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 02
> **Guided lab coverage:** Part 1 Step 1.3 (`extract_raw_data`: PERMISSIVE read, record-count check) and Step 1.4 (`tag_and_route`: boolean quality flags, `_is_rejected`, `rejection_reason`, clean/rejected routing)

## Learning objectives

By the end of this lab you can:

- Explain why a pipeline should **fail** when far too few records arrive (a volume check).
- Build boolean **quality flags** with `withColumn`, and combine them into one rejection decision.
- Explain SQL's three-valued logic (`TRUE` / `FALSE` / `NULL`) and the bug it causes in filters.
- Turn flags into a human-readable `rejection_reason` with `when(...).otherwise(...)`.
- Prove that every row is accounted for: `total = clean + rejected`.

## Steps

### Step 1 — Add `extract_raw_data()`

Append to `C:\DataEngineering\sk03\jobs\etl_job.py` (guided lab Step 1.3):

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

- **The path** `raw/orders/dt=<process_date>/` is built from the date argument — never from "today". That is what makes re-runs and backfills possible (Lab 06).
- **`.cache()`** keeps the parsed rows in memory: the next steps count and filter them several times, and Spark only allows `_corrupt_record` queries on cached data (Lab 02 Step 7).
- **The volume check.** If the source export breaks and only 50 lines arrive, a pipeline that "succeeds" with 50 rows is worse than one that fails — dashboards would show a catastrophic sales drop that never happened. `MIN_RECORD_COUNT` turns that silent problem into a loud failure.

**Expected output:** none yet.

✅ **Checkpoint:** `etl_job.py` contains `def extract_raw_data`.

### Step 2 — Try the extract, then trip the volume check

```powershell
docker exec -it airflow-scheduler python
```

```python
import sys; sys.path.insert(0, "/opt/airflow/jobs")
from pyspark.sql import functions as F
import etl_job
from etl_job import PipelineConfig, create_spark_session, extract_raw_data
spark = create_spark_session("lab03")

raw = extract_raw_data(spark, "2026-07-01")

PipelineConfig.MIN_RECORD_COUNT = 5_000          # pretend we expect 5,000 lines
try:
    extract_raw_data(spark, "2026-07-01")
except ValueError as e:
    print("ABORTED:", e)
PipelineConfig.MIN_RECORD_COUNT = 1_000          # put it back
```

**Expected output** (log timestamps differ):

```
2026-..-.. ..:..:..,... [INFO] Reading raw data from: s3a://urbangear-data-lake/raw/orders/dt=2026-07-01/
2026-..-.. ..:..:..,... [INFO] [EXTRACT] 1,968 records read
2026-..-.. ..:..:..,... [INFO] Reading raw data from: s3a://urbangear-data-lake/raw/orders/dt=2026-07-01/
2026-..-.. ..:..:..,... [INFO] [EXTRACT] 1,968 records read
ABORTED: Record count 1,968 is below the minimum threshold 5,000. Source export may have failed. Aborting pipeline.
```

Keep the session open.

✅ **Checkpoint:** you saw `ABORTED: Record count 1,968 is below the minimum threshold 5,000`.

### Step 3 — Understand the NULL trap before writing any flag

In SQL and Spark, a comparison with `NULL` is neither true nor false — it is `NULL`. Run this in the same session:

```python
demo = spark.createDataFrame([(1, 5), (2, -1), (3, None)], "id INT, quantity INT")
flagged = demo.withColumn("negative", F.col("quantity") <= 0)
flagged.show()
print("kept as clean:   ", [r.id for r in flagged.filter(~F.col("negative")).collect()])
print("sent to rejected:", [r.id for r in flagged.filter(F.col("negative")).collect()])
```

**Expected output:**

```
+---+--------+--------+
| id|quantity|negative|
+---+--------+--------+
|  1|       5|   false|
|  2|      -1|    true|
|  3|    NULL|    NULL|
+---+--------+--------+

kept as clean:    [1]
sent to rejected: [2]
```

Row 3 **vanished** — a filter keeps a row only when the condition is `TRUE`, and `NOT NULL` is still `NULL`. The same thing happens with `OR`: `NULL OR FALSE` is `NULL`. That is why our flags wrap the comparison in `F.coalesce(<flag>, F.lit(False))` — every flag is always `TRUE` or `FALSE`, and every row lands in exactly one output. (A missing platform is treated as *invalid*, so that flag defaults to `True`.)

✅ **Checkpoint:** you can explain why row 3 disappeared.

### Step 4 — Add `tag_and_route()`

Append to `etl_job.py` (guided lab Step 1.4):

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
```

| Flag | TRUE when | Why it matters |
|---|---|---|
| `_null_critical` | `order_id`, `customer_id`, `order_date`, `quantity` or `unit_price` is NULL | Can't attribute, date or price the sale |
| `_negative_qty` | `quantity <= 0` | Impossible sale (returns are a separate flag) |
| `_future_date` | `order_date` is after "now" | Clock/system error at the source |
| `_invalid_platform` | `lower(trim(platform))` not in `VALID_PLATFORMS` | Unknown sales channel. We **normalise first** so `" Website "` counts as valid |
| `_corrupt_json` | `_corrupt_record` is not NULL | The line wasn't JSON at all |

- **Flags, not filters.** Each rule becomes a visible column, so the rejected output shows *exactly* which rule fired — essential when someone asks "why is this order missing?".
- **`rejection_reason`** — `when(...).when(...).otherwise(None)` picks the **first** matching rule, in priority order (a corrupt line also has NULL fields; we report the root cause).
- **Routing** — the same `_is_rejected` column drives both filters, so a row can never be in both outputs, or neither.

**Expected output:** none yet.

✅ **Checkpoint:** `etl_job.py` contains `def tag_and_route`.

### Step 5 — Route one day and inspect the rejects

Python caches modules, so reload `etl_job` to pick up the new function, then run it:

```python
import importlib; importlib.reload(etl_job)
from etl_job import tag_and_route

clean, rejected = tag_and_route(raw, "2026-07-01")
total = raw.count()
print("total", total, "| clean + rejected", clean.count() + rejected.count())
rejected.groupBy("rejection_reason").count().orderBy("rejection_reason").show()
clean.select(F.lower(F.trim("platform")).alias("platform")).groupBy("platform").count().orderBy("platform").show()
```

**Expected output:**

```
2026-..-.. ..:..:..,... [INFO] [QUALITY] Clean: 1,909 | Rejected: 59
total 1968 | clean + rejected 1968
+-------------------+-----+
|   rejection_reason|count|
+-------------------+-----+
|       corrupt_json|   15|
|  future_order_date|   15|
|   invalid_platform|   15|
|  negative_quantity|    2|
|null_critical_field|   12|
+-------------------+-----+

+-----------+-----+
|   platform|count|
+-----------+-----+
|marketplace|  609|
| mobile_app|  591|
|    website|  709|
+-----------+-----+
```

- **Reconciliation:** 1,909 + 59 = 1,968. Every line is accounted for — the single most important property of a batch job.
- The messy `" Website "` rows were *kept* (normalised); only `fax` and the `NULL` platforms of corrupt lines were rejected.

✅ **Checkpoint:** `Clean: 1,909 | Rejected: 59` and the five reason counts match.

### Step 6 — Compare all five days

```python
for d in ["2026-07-01", "2026-07-02", "2026-07-03", "2026-07-04", "2026-07-05"]:
    r = extract_raw_data(spark, d)
    c, j = tag_and_route(r, d)
    print(d, "total", r.count(), "clean", c.count(), "rejected", j.count(),
          f"rate {j.count() / r.count():.2%}")
    r.unpersist()
spark.stop()
```

**Expected output** (log lines omitted):

```
2026-07-01 total 1968 clean 1909 rejected 59 rate 3.00%
2026-07-02 total 1989 clean 1924 rejected 65 rate 3.27%
2026-07-03 total 2004 clean 1950 rejected 54 rate 2.69%
2026-07-04 total 2000 clean 1933 rejected 67 rate 3.35%
2026-07-05 total 2004 clean 1768 rejected 236 rate 11.78%
```

The four normal days sit around 3%. 2026-07-05 — the "bad day" you generated with `--dirty_rate 0.12` — is at 11.78%. In Lab 07 the Airflow **quality gate** compares this rate with `MAX_REJECT_RATE` (5%) and refuses to load that day.

Type `exit()`.

✅ **Checkpoint:** four days at ~3%, one day at 11.78%.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AttributeError: module 'etl_job' has no attribute 'tag_and_route'` | Python cached the old module | `importlib.reload(etl_job)` or restart the Python session |
| `clean + rejected` is smaller than `total` | A flag can be NULL (missing `coalesce`) | Compare your flags with Step 4 — every comparison is wrapped in `F.coalesce` |
| `AnalysisException ... _corrupt_record` | Data not cached | `extract_raw_data` must end its read with `.cache()` |
| `invalid_platform` count is ~200, not 15 | Validation done on the raw value | Validate `F.lower(F.trim(F.col("platform")))`, not `F.col("platform")` |
| `future_order_date` count is 0 | `order_date` read as string | `order_date` must be `TimestampType()` in `RAW_ORDER_SCHEMA` |
| `ABORTED: Record count ...` on a normal day | `MIN_RECORD_COUNT` left at 5,000 | Restore `1_000` (or start a new session) |

## Key takeaways

- `etl_job.py` now has the guided lab's Steps 1.3–1.4: `extract_raw_data()` and `tag_and_route()`.
- A volume check (`MIN_RECORD_COUNT`) converts "the export broke" from a silent success into a loud failure.
- Flags are columns, so every rejection is explainable; `rejection_reason` names the first rule that fired.
- NULLs break boolean logic: wrap flags in `coalesce`, or rows silently disappear from *both* outputs.
- Always reconcile: `total = clean + rejected` (1,968 = 1,909 + 59 for 2026-07-01).

## Next

[Lab 04 — Transformations, Window Functions and Partitioned Writes](Lab_04_Transformations_Windows_and_Partitioned_Writes.md): finish `etl_job.py` — revenue maths, window functions, quality metrics, partitioned Parquet with dynamic overwrite, and the `--process_date` entry point.
