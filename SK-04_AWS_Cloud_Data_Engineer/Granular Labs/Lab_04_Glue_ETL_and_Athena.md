# Lab_04 — Glue ETL & Athena

> **Time:** 4–6 hours | **Cost: $1–3** — the first lab with real (small) spend.
>
> **Cost Check:**
> - **Glue crawler:** billed per second, ~$0.44/DPU-hour, 10-minute minimum → each crawl of our tiny data ≈ **$0.07**.
> - **Glue Spark job:** we run at the 2-DPU minimum with a 10-minute timeout → each run ≈ **$0.15**. A forgotten *schedule* is the danger — we create none.
> - **Athena:** $5 per TB scanned, 10 MB minimum per query → our queries ≈ **$0.00005 each**. Athena has no idle cost.
> - **What bills while idle: nothing** — Catalog storage is free at our scale, jobs/crawlers only bill when running. The trap is scheduled crawlers/jobs; we schedule nothing.

---

## 1. Environment Setup

Verify prior state (Lab_03 must be applied):

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws\terraform\lake
terraform plan     # Expected: No changes.
aws s3 ls s3://swifthaul-francis-raw/telemetry/ --recursive   # sample CSVs present (Lab_03 Step 9)
aws iam get-role --role-name swifthaul-glue-role --query Role.Arn   # from Lab_01
```

Nothing to install — Glue and Athena are serverless (AWS runs the compute; you bring code and pay per use). New terms: **AWS Glue** — managed Spark ETL plus a metadata catalog; **Glue Data Catalog** — a Hive-compatible "database of table definitions" describing files in S3; **crawler** — a Glue robot that scans S3 and writes table definitions; **Athena** — serverless SQL engine (Presto/Trino under the hood) that queries S3 files using Catalog definitions; **Parquet** — columnar file format (you met it in SK-03) that makes analytics fast and cheap; **DPU** — Data Processing Unit, Glue's billing unit (4 vCPU + 16 GB).

Add one more day of sample data so the job has something to dedupe and clean — create `telemetry_atlanta_2026-07-07.csv` containing a **duplicate row** and a **bad row** (negative fuel):

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws\sample-data
@'
truck_id,ts,lat,lon,speed_kmh,fuel_pct,engine_temp_c
T1001,2026-07-07T20:55:01Z,33.749,-84.388,45,61,90
T1001,2026-07-07T20:55:01Z,33.749,-84.388,45,61,90
T1004,2026-07-07T20:58:44Z,33.700,-84.300,88,-12,93
'@ | Out-File -Encoding utf8 telemetry_atlanta_2026-07-07.csv
aws s3 cp telemetry_atlanta_2026-07-07.csv s3://swifthaul-francis-raw/telemetry/depot=atlanta/year=2026/month=07/day=07/
```

---

## 2. Business Context

SwiftHaul's analysts currently download CSVs and pivot them in Excel — slow, error-prone, no dedup, no validation. The client wants: nightly raw files automatically cleaned, deduplicated, quality-checked, converted to a fast format, and queryable in SQL within the hour.

This raw→processed hop is the beating heart of nearly every batch platform in industry (the "T" in ELT/ETL). Consumers: analysts via Athena/BI tools, the warehouse (Lab_05), data scientists. If it fails: dashboards show yesterday's numbers (trust erodes), duplicated rows inflate KPIs (a real logistics firm once double-billed customers because a re-uploaded file was processed twice), or bad rows (negative fuel) silently poison downstream models.

---

## 3. Concept Explanation

**Why a catalog at all?** Files in S3 are just bytes. The Data Catalog stores *schema* (columns, types, partitions, location), so every engine — Athena, Glue, Redshift Spectrum, EMR — shares one definition. Alternatives: Hive Metastore (self-managed, what SK-03 tooling used implicitly), Unity Catalog (Databricks). Crawler vs hand-written DDL: crawlers infer schema automatically (fast, occasionally wrong — everything-is-a-string problems); DDL is explicit and reviewable. Pros team: crawl once to bootstrap, then own the DDL. We'll do exactly that mindset-wise: crawl, then *inspect and correct*.

**Why Glue for ETL?** It's serverless Spark: no cluster to manage (unlike the SK-03 local Spark or EMR), per-second billing, tight Catalog integration. Cons: cold starts (~1 min), less control, cost surprises via schedules. Alternatives: EMR (more control), Lambda (small data only), Databricks.

**Job bookmarks** = Glue's built-in incremental processing: it remembers which files it has already processed, so re-running only handles *new* files. That's idempotency-by-construction for file-based pipelines — and your project's late-arrival handler.

**Why Parquet again (SK-03 recap):** columnar layout → engines read only needed columns; typed + compressed → ~10x smaller than CSV; both together slash Athena's per-TB-scanned bill.

**Athena's cost model is the design constraint:** $5/TB *scanned*. Partition pruning + Parquet + column selection are how you turn a $50 query into a $0.005 query. **CTAS** (`CREATE TABLE AS SELECT`) lets Athena itself write transformed Parquet — a lightweight alternative to Glue jobs for SQL-shaped transforms.

---

## 4. Step-by-Step Implementation

We extend the Lab_03 Terraform folder — infrastructure as code, always.

### Step 1 — Terraform: Catalog database, crawler, and Athena results bucket

Append to `main.tf` (in `terraform\lake\`):

```hcl
resource "aws_glue_catalog_database" "swifthaul" {
  name = "swifthaul_${var.owner}"
}

resource "aws_glue_crawler" "raw_telemetry" {
  name          = "swifthaul-raw-telemetry"
  role          = "swifthaul-glue-role"          # created in Lab_01
  database_name = aws_glue_catalog_database.swifthaul.name
  s3_target {
    path = "s3://${aws_s3_bucket.zone["raw"].bucket}/telemetry/"
  }
  # NOTE: no schedule block — on-demand only. A schedule is a cost leak.
}

resource "aws_s3_bucket" "athena_results" {
  bucket = "swifthaul-${var.owner}-athena-results"
}

resource "aws_athena_workgroup" "swifthaul" {
  name = "swifthaul"
  configuration {
    result_configuration {
      output_location = "s3://${aws_s3_bucket.athena_results.bucket}/"
    }
    bytes_scanned_cutoff_per_query = 1073741824   # 1 GB hard cap per query
  }
}
```

```powershell
terraform apply   # Expected: 4 to add
```

**Why `bytes_scanned_cutoff_per_query`?** Athena kills any query that would scan >1 GB — a seatbelt against accidental full-lake scans. **Why a results bucket?** Athena must write query outputs somewhere; it refuses to run without one.
**Troubleshooting:** crawler creation fails with role errors → the Lab_01 role's `AWSGlueServiceRole` policy needs S3 read on your buckets; your Lab_01 policy names cover `swifthaul-*` — if not, attach `AmazonS3ReadOnlyAccess` to `swifthaul-glue-role` temporarily.

### Step 2 — Run the crawler (on demand)

```powershell
aws glue start-crawler --name swifthaul-raw-telemetry
aws glue get-crawler --name swifthaul-raw-telemetry --query "Crawler.State"
# Poll until READY (takes 1–3 minutes)
aws glue get-tables --database-name swifthaul_francis --query "TableList[].Name"
```

**Expected:** a table named `telemetry`. Inspect it:

```powershell
aws glue get-table --database-name swifthaul_francis --name telemetry --query "Table.{Cols:StorageDescriptor.Columns,Parts:PartitionKeys}"
```

**Verify:** columns `truck_id…engine_temp_c` plus partition keys `depot, year, month, day` — the crawler read your Hive-style key names, exactly as promised in Lab_02.
**Common mistake:** table shows one weird column → your CSVs lack a consistent header or delimiters differ; fix the files, delete the table, re-crawl.

### Step 3 — First Athena queries

Console → Athena → Query editor → workgroup **swifthaul** → database `swifthaul_francis`:

```sql
SELECT depot, count(*) AS rows, min(fuel_pct) AS min_fuel
FROM telemetry
GROUP BY depot;
```

**Expected:** 3–4 depots, and `min_fuel = -12` for atlanta — your planted bad row, visible in SQL. Note "Data scanned" at the bottom: a few KB. Now prove partition pruning:

```sql
SELECT * FROM telemetry WHERE depot = 'atlanta' AND day = '07';
```

**Verify:** scanned bytes drop — Athena skipped every non-matching partition prefix. This is the cost lever.
**Troubleshooting:** zero rows → crawler ran before your day=07 upload; re-run crawler (new partitions need cataloging — or `MSCK REPAIR TABLE telemetry;`).

### Step 4 — Write the Glue ETL script

Create `glue-scripts\clean_telemetry.py` in your project folder:

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql import functions as F

args = getResolvedOptions(sys.argv, ["JOB_NAME", "raw_path", "processed_path"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

df = (spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv(args["raw_path"]))

clean = (df
    .dropDuplicates(["truck_id", "ts"])                      # dedupe re-sent rows
    .filter((F.col("fuel_pct") >= 0) & (F.col("fuel_pct") <= 100))
    .filter(F.col("speed_kmh").between(0, 160))
    .withColumn("ts", F.to_timestamp("ts"))
    .withColumn("year", F.year("ts"))
    .withColumn("month", F.format_string("%02d", F.month("ts")))
    .withColumn("day", F.format_string("%02d", F.dayofmonth("ts"))))

print(f"rows_in={df.count()} rows_out={clean.count()}")       # goes to CloudWatch Logs

(clean.write
    .mode("overwrite")
    .partitionBy("year", "month", "day")
    .parquet(args["processed_path"]))

job.commit()   # commits the job bookmark
```

Read it: dedupe on business key, reject impossible values, derive partition columns, write partitioned Parquet, log row counts, commit the bookmark. Upload it (Glue reads scripts from S3):

```powershell
aws s3 cp glue-scripts\clean_telemetry.py s3://swifthaul-francis-processed/scripts/
```

### Step 5 — Terraform: the Glue job (cost-guarded)

Append to `main.tf`:

```hcl
resource "aws_glue_job" "clean_telemetry" {
  name         = "swifthaul-clean-telemetry"
  role_arn     = "arn:aws:iam::<ACCOUNT_ID>:role/swifthaul-glue-role"
  glue_version = "4.0"
  number_of_workers = 2
  worker_type       = "G.1X"
  timeout           = 10          # minutes — hard cost cap
  command {
    script_location = "s3://${aws_s3_bucket.zone["processed"].bucket}/scripts/clean_telemetry.py"
    python_version  = "3"
  }
  default_arguments = {
    "--raw_path"                = "s3://${aws_s3_bucket.zone["raw"].bucket}/telemetry/"
    "--processed_path"          = "s3://${aws_s3_bucket.zone["processed"].bucket}/telemetry/"
    "--job-bookmark-option"     = "job-bookmark-enable"
    "--enable-metrics"          = "true"
    "--enable-continuous-cloudwatch-log" = "true"
  }
}
```

Replace `<ACCOUNT_ID>`; `terraform apply`. **The two cost guards:** 2 workers (the near-minimum) and `timeout = 10` — a runaway job dies at ~$0.15, not $150.

### Step 6 — Run and verify the job

```powershell
aws glue start-job-run --job-name swifthaul-clean-telemetry
aws glue get-job-runs --job-name swifthaul-clean-telemetry --query "JobRuns[0].{State:JobRunState,Min:ExecutionTime}"
# Poll until SUCCEEDED (3–8 min including cold start)
aws s3 ls s3://swifthaul-francis-processed/telemetry/ --recursive
```

**Expected:** Parquet files under `year=2026/month=07/day=06/` and `day=07/`. The duplicate T1001 row and the negative-fuel row are gone — verify next step.
**Troubleshooting:** `AccessDenied` writing to processed → the Glue role lacks `s3:PutObject` on the processed bucket; add it. Failed with `Module not found` → script path typo in `script_location`.

### Step 7 — Catalog processed data and prove the cleaning worked

Add a second crawler for `s3://…-processed/telemetry/` in Terraform (copy the first block, rename), apply, run it, then in Athena:

```sql
SELECT count(*) FROM telemetry_processed;          -- Expected: rows_in - 2 (one dup, one bad row removed)
SELECT min(fuel_pct) FROM telemetry_processed;     -- Expected: >= 0
```

Compare bytes scanned vs the CSV table: Parquet scans less for the same answer. Screenshot both — this is Project-style evidence.

### Step 8 — Prove job bookmarks (incremental idempotency)

Run the job again with **no new files**:

```powershell
aws glue start-job-run --job-name swifthaul-clean-telemetry
```

Check the run's CloudWatch log (preview of Lab_06): `rows_in=0` — the bookmark skipped already-processed files. Upload one new CSV, run again: only that file processes. **This is the late-arrival mechanism your Project relies on.**

### Step 9 — CTAS: curated layer via pure SQL

In Athena:

```sql
CREATE TABLE depot_daily_summary
WITH (external_location = 's3://swifthaul-francis-curated/depot_daily_summary/',
      format = 'PARQUET') AS
SELECT depot, year, month, day,
       count(*) AS readings,
       avg(speed_kmh) AS avg_speed,
       avg(fuel_pct)  AS avg_fuel
FROM telemetry_processed
GROUP BY depot, year, month, day;
```

**Verify:** `SELECT * FROM depot_daily_summary;` returns per-depot daily rows, and Parquet files appear in the curated bucket. This table feeds Redshift (Lab_05) and Power BI (Project).

---

## 5. Production Engineering Practices

- **Idempotency:** bookmarks (skip processed input) + `mode("overwrite")` per partition (re-runs replace, not append) = safe re-runs. *Failure story:* a team without bookmarks re-ran a month of jobs after an outage; every metric doubled and the CFO presented wrong numbers to the board.
- **Validation with evidence:** the `rows_in/rows_out` log line is primitive but mighty — a sudden 40% drop in rows_out is your first data-quality alarm (you'll alert on it in Lab_06). Production teams graduate to Glue Data Quality or Great Expectations.
- **Reject, don't delete:** we filtered bad rows away; production pipelines write them to a `quarantine/` prefix with a reason column so nothing silently vanishes. The Project requires this.
- **Config over hardcoding:** paths arrive as job arguments — the same script serves dev and prod.
- **Cost guards as code:** worker count, timeout, no schedules, Athena scan cutoff — all in Terraform, all reviewable.

---

## 6. Reflection

**What you learned:** Data Catalog, crawlers vs DDL, serverless Spark ETL, bookmarks, Parquet economics, Athena partition pruning, CTAS.

**Interview questions:**

1. *What is the Glue Data Catalog?* — A managed Hive-compatible metastore mapping S3 data to table schemas, shared by Athena/Glue/Redshift Spectrum/EMR.
2. *Crawler vs hand-written DDL?* — Crawlers infer (fast, can mistype); DDL is explicit and reviewable. Common pattern: crawl to bootstrap, then own the DDL.
3. *How does Athena charge, and how do you cut the bill?* — $5/TB scanned; partition pruning, columnar formats, compression, selecting only needed columns, workgroup scan caps.
4. *What are Glue job bookmarks?* — Per-job state tracking processed input files, enabling incremental, re-run-safe file pipelines.
5. *How do you make a batch ETL idempotent?* — Deterministic partition overwrites, bookmarks/watermarks on input, dedup on business keys, no blind appends.
6. *Parquet vs CSV for a lake?* — Columnar, typed, compressed, splittable, predicate pushdown → faster and ~100x cheaper to scan; CSV only survives as an ingest format.
7. *When Athena CTAS vs a Glue job?* — CTAS for SQL-expressible transforms of cataloged data (cheap, fast); Glue for complex logic, non-SQL cleaning, bookmarks, external libraries.
8. *A depot re-uploads yesterday's file with a new name — what happens in your pipeline?* — Bookmark sees a new file and processes it, but `dropDuplicates` on (truck_id, ts) removes the repeated rows, and partition overwrite keeps output clean.

**Trap:** "Athena is free when idle, so the lake costs nothing, right?" — Storage bills, Catalog beyond free tier bills, and one careless `SELECT *` over years of CSV can cost real money; workgroup cutoffs exist for a reason.

**Key takeaways:** the catalog is the contract between storage and every engine; scan-based pricing makes format+partitioning a cost decision; bookmarks + overwrite = idempotent batch; guards (timeouts, cutoffs) belong in code.

---

## Teardown

Glue and Athena bill only when running, so end-of-session teardown = **make sure nothing is scheduled or running**:

```powershell
aws glue get-crawler --name swifthaul-raw-telemetry --query "Crawler.State"   # READY = not running
aws glue get-job-runs --job-name swifthaul-clean-telemetry --query "JobRuns[?JobRunState=='RUNNING']"  # Expected: []
```

Keep everything for Lab_05/06 (idle cost: S3 pennies). Full module teardown remains `terraform destroy` after emptying buckets (Lab_03 Teardown), plus in Athena: `DROP TABLE depot_daily_summary;` (CTAS files are deleted with the curated bucket). Verify with `aws resourcegroupstaggingapi get-resources --region eu-west-1`.

**Next:** [Lab_05_Redshift_Serverless_Warehouse.md](Lab_05_Redshift_Serverless_Warehouse.md) — the one lab where idle resources can genuinely cost money. Read its Cost Check twice.
