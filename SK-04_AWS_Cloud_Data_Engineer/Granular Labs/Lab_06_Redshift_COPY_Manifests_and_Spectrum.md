# Lab 06 — Loading Redshift: COPY, Manifests and Spectrum

> **Goal:** Load the Firehose Parquet files into Redshift idempotently (manifest → COPY into staging → delete + insert in one transaction), diagnose load errors, and query the same S3 data in place with Redshift Spectrum.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 05 (cluster + schema) and Lab 04 (Parquet files in `raw/events/`, Glue table, IAM roles)
> **Cost:** ~$3–4 — the cluster bills ~$1.09/hour while running; Spectrum bills $5/TB scanned (fractions of a cent here). **Delete the cluster in Teardown.**
> **Guided lab coverage:** PART 4 — Step 4.3 (manifest, COPY, staging, transaction, `sys_load_history`, `stl_load_errors`) and Step 4.4 (partition registration, Spectrum external schema, hot/cold query)

## Learning objectives

By the end of this lab you can:

- Write a COPY **manifest** (including the `content_length` that Parquet requires) and explain why manifests make loads auditable.
- Load Parquet with `COPY … FORMAT AS PARQUET MANIFEST IAM_ROLE default` into a staging table.
- Make a daily load idempotent with delete-then-insert inside **one transaction** (`batch_execute_statement`).
- Read `sys_load_history` and `stl_load_errors` to verify or debug a load.
- Register Glue partitions and query S3 through a Spectrum external schema next to local tables.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Get the cluster ready and find your data day

If you paused the cluster in Lab 05:

```bash
aws redshift resume-cluster --cluster-identifier "$CLUSTER_ID" --query 'Cluster.ClusterStatus'
aws redshift wait cluster-available --cluster-identifier "$CLUSTER_ID"
```

If you deleted it: `python scripts/07_create_cluster.py && python scripts/rs.py sql/01_schema.sql` (~15 minutes).

Then find the day Firehose wrote in Lab 04:

```bash
aws s3 ls "s3://$BUCKET/raw/events/" --recursive
```

**Expected output:**

```
2026-03-19 14:10:12     201234 raw/events/year=2026/month=03/day=19/hour=14/cloudmetrics-events-to-s3-1-2026-03-19-14-05-12-1a2b3c4d-...
```

✅ **Checkpoint:** you know your **data day** (here `2026-03-19`) and the cluster is `available`. No files? Repeat Lab 04 Steps 2, 5 and 6, then come back.

### Step 2 — What a manifest is

A **manifest** is a JSON file listing exactly which S3 objects one COPY loads. Loading a *prefix* instead would also pick up files that arrive mid-load; the manifest freezes the file list and stays in S3 as an audit record ("which files built the 19 March numbers?"). For Parquet and ORC, each entry **must** include `meta.content_length` (the object size in bytes):

```json
{
  "entries": [
    {
      "url": "s3://cloudmetrics-data-lake-francis/raw/events/year=2026/month=03/day=19/hour=14/cloudmetrics-events-to-s3-1-2026-03-19-14-05-12-1a2b3c4d",
      "mandatory": true,
      "meta": {"content_length": 201234}
    }
  ]
}
```

`"mandatory": true` makes COPY fail if the file is missing, instead of silently loading less.

**Expected output:** none — this step is reading.

✅ **Checkpoint:** you can explain why `mandatory: true` protects the row count.

### Step 3 — The daily load script: `scripts/08_load_day.py`

Read it in three parts:

1. **Build the manifest** by listing the day's `raw/events/year=/month=/day=/` objects (with their sizes) and upload it to `manifests/`.
2. **One transaction** (`run_transaction` → `batch_execute_statement`): empty `staging.api_events_raw`, `COPY` the manifest into it (Parquet maps columns **by position**, which is why staging matches the Glue table exactly), delete the day from `analytics.fact_api_events`, insert it again with `event_date = TRUNC(event_timestamp)`. If any statement fails, everything rolls back.
3. **Verify** with the row count, `sys_load_history` and `stl_load_errors`.

```python
# scripts/08_load_day.py   usage: python scripts/08_load_day.py [YYYY-MM-DD]   (default: today, UTC)
import json
import sys
from datetime import datetime, timezone

import boto3

from cm_config import BUCKET, REGION
from rs import run, run_transaction, show

# Idempotent daily load: delete then insert
process_date = sys.argv[1] if len(sys.argv) > 1 else datetime.now(timezone.utc).strftime("%Y-%m-%d")
datetime.strptime(process_date, "%Y-%m-%d")          # reject anything that is not a date
year, month, day = process_date.split("-")
prefix = f"raw/events/year={year}/month={month}/day={day}/"

# Step 1: Build the manifest from the files Firehose wrote for that day
# (content_length is REQUIRED in manifests for Parquet/ORC COPY)
s3 = boto3.client("s3", region_name=REGION)
entries = []
for page in s3.get_paginator("list_objects_v2").paginate(Bucket=BUCKET, Prefix=prefix):
    for obj in page.get("Contents", []):
        entries.append({
            "url": f"s3://{BUCKET}/{obj['Key']}",
            "mandatory": True,
            "meta": {"content_length": obj["Size"]},
        })
if not entries:
    sys.exit(f"No files under s3://{BUCKET}/{prefix} — run the producer and wait 5 minutes for Firehose")

manifest_key = f"manifests/events_{process_date}.manifest"
s3.put_object(Bucket=BUCKET, Key=manifest_key,
              Body=json.dumps({"entries": entries}, indent=2).encode("utf-8"))
print(f"Manifest s3://{BUCKET}/{manifest_key} lists {len(entries)} file(s)")

# Step 2: COPY into staging, then replace the day in the fact table — ONE transaction
run_transaction([
    "DELETE FROM staging.api_events_raw",
    f"""COPY staging.api_events_raw
        FROM 's3://{BUCKET}/{manifest_key}'
        IAM_ROLE default
        FORMAT AS PARQUET
        MANIFEST
        COMPUPDATE ON
        STATUPDATE ON""",
    f"DELETE FROM analytics.fact_api_events WHERE event_date = '{process_date}'::DATE",
    f"""INSERT INTO analytics.fact_api_events (
            event_id, customer_id, event_type, endpoint, method,
            status_code, response_time_ms, request_size_bytes,
            response_size_bytes, event_timestamp, region, event_date)
        SELECT event_id, customer_id, event_type, endpoint, method,
               status_code, response_time_ms, request_size_bytes,
               response_size_bytes, event_timestamp, region, TRUNC(event_timestamp)
        FROM staging.api_events_raw
        WHERE TRUNC(event_timestamp) = '{process_date}'::DATE""",
])

# Step 3: Verify — row count, load history, and load errors
show(run(f"SELECT COUNT(*) AS rows_loaded FROM analytics.fact_api_events "
         f"WHERE event_date = '{process_date}'::DATE"))
show(run("SELECT query_id, table_name, status, loaded_rows, start_time "
         "FROM sys_load_history ORDER BY start_time DESC LIMIT 5"))
show(run("""SELECT filename, colname, type, raw_field_value, err_reason
            FROM stl_load_errors
            WHERE starttime > GETDATE() - INTERVAL '1 hour'
            ORDER BY starttime DESC
            LIMIT 20"""))
print(f"Load complete for {process_date}")
```

Run it for **your** data day:

```bash
python scripts/08_load_day.py 2026-03-19
aws s3 cp "s3://$BUCKET/manifests/events_2026-03-19.manifest" -
```

**Expected output:**

```
Manifest s3://cloudmetrics-data-lake-francis/manifests/events_2026-03-19.manifest lists 1 file(s)
rows_loaded
10000
query_id | table_name | status | loaded_rows | start_time
1234567 | api_events_raw | completed | 10000 | 2026-03-19 15:02:11.482113
Load complete for 2026-03-19
{
  "entries": [
    {
      "url": "s3://cloudmetrics-data-lake-francis/raw/events/year=2026/month=03/day=19/hour=14/...",
      "mandatory": true,
      "meta": {
        "content_length": 201234
      }
    }
  ]
}
```

✅ **Checkpoint:** 10,000 rows for the day (the number you sent in Lab 04), and `stl_load_errors` printed nothing.

### Step 4 — Prove idempotency

Run the same load again:

```bash
python scripts/08_load_day.py 2026-03-19
```

**Expected output:** the same `rows_loaded` — **10000**, not 20000.

✅ **Checkpoint:** you can explain why: the transaction deletes the day before inserting it. Re-running a failed or repeated job can never double-count — the property every warehouse load needs.

### Step 5 — Read a load error

Parquet COPY errors fail the whole statement (`rs.py` raises them with Redshift's message). For text formats such as CSV, `MAXERROR` lets COPY skip bad rows and record each one in **`stl_load_errors`**. Practise reading it with a tiny CSV — 2 good rows, 1 too-long value. Save `scripts/lab06_load_errors.py`:

```python
# scripts/lab06_load_errors.py — see stl_load_errors in action (2 good rows, 1 bad)
import boto3

from cm_config import BUCKET, REGION
from rs import run, show

csv_text = (
    "customer_id,company_name,plan_type\n"
    "cust-0001,Acme Corp,enterprise\n"
    "cust-0002,Globex,pro\n"
    "cust-0003,Initech,this-plan-name-is-much-longer-than-the-fifty-characters-the-column-allows\n"
)
key = "temp/lab06/customers.csv"
boto3.client("s3", region_name=REGION).put_object(Bucket=BUCKET, Key=key, Body=csv_text.encode("utf-8"))

run("CREATE TABLE IF NOT EXISTS staging.customers_practice "
    "(customer_id VARCHAR(36), company_name VARCHAR(200), plan_type VARCHAR(50))")
run("DELETE FROM staging.customers_practice")
run(f"COPY staging.customers_practice FROM 's3://{BUCKET}/{key}' "
    "IAM_ROLE default FORMAT AS CSV IGNOREHEADER 1 MAXERROR 5")
show(run("SELECT COUNT(*) AS rows_loaded FROM staging.customers_practice"))
show(run("SELECT line_number, TRIM(colname) AS colname, TRIM(err_reason) AS err_reason "
         "FROM stl_load_errors ORDER BY starttime DESC LIMIT 1"))
run("DROP TABLE staging.customers_practice")
```

```bash
python scripts/lab06_load_errors.py
```

**Expected output:**

```
rows_loaded
2
line_number | colname | err_reason
4 | plan_type | String length exceeds DDL length
```

✅ **Checkpoint:** you can find *which line*, *which column* and *why* a row was rejected — and you know that a COPY reporting success can still have skipped rows when `MAXERROR` > 0, so always reconcile counts.

### Step 6 — Register partitions for Spectrum: `scripts/09_register_partitions.py`

**Redshift Spectrum** queries files in S3 through Glue Catalog tables — no loading. But it only reads **registered partitions**, and Firehose creates new `year=/month=/day=/hour=` folders without registering them. This script finds every hour folder and registers it with `batch_create_partition` (already-registered ones are skipped):

```python
# scripts/09_register_partitions.py — register every Firehose hour folder as a Glue partition
import re

import boto3

from cm_config import BUCKET, GLUE_DATABASE, GLUE_TABLE, REGION

glue = boto3.client("glue", region_name=REGION)
s3 = boto3.client("s3", region_name=REGION)

table_sd = glue.get_table(DatabaseName=GLUE_DATABASE, Name=GLUE_TABLE)["Table"]["StorageDescriptor"]
pattern = re.compile(r"^raw/events/year=(\d{4})/month=(\d{2})/day=(\d{2})/hour=(\d{2})/")

found = set()
for page in s3.get_paginator("list_objects_v2").paginate(Bucket=BUCKET, Prefix="raw/events/"):
    for obj in page.get("Contents", []):
        match = pattern.match(obj["Key"])
        if match:
            found.add(match.groups())

partitions = []
for year, month, day, hour in sorted(found):
    sd = dict(table_sd)
    sd["Location"] = f"s3://{BUCKET}/raw/events/year={year}/month={month}/day={day}/hour={hour}/"
    partitions.append({"Values": [year, month, day, hour], "StorageDescriptor": sd})

for i in range(0, len(partitions), 100):            # API limit: 100 partitions per call
    response = glue.batch_create_partition(DatabaseName=GLUE_DATABASE, TableName=GLUE_TABLE,
                                           PartitionInputList=partitions[i:i + 100])
    for error in response.get("Errors", []):
        if error["ErrorDetail"]["ErrorCode"] != "AlreadyExistsException":
            print("Partition error:", error["ErrorDetail"])

print(f"{len(partitions)} partition(s) registered in {GLUE_DATABASE}.{GLUE_TABLE}")
```

```bash
python scripts/09_register_partitions.py
aws glue get-partitions --database-name "$GLUE_DATABASE" --table-name "$GLUE_TABLE" --query 'Partitions[].Values'
```

**Expected output:**

```
1 partition(s) registered in cloudmetrics_raw.events
[
    [
        "2026",
        "03",
        "19",
        "14"
    ]
]
```

✅ **Checkpoint:** one partition per hour folder you saw in Step 1. In production this runs after every Firehose delivery window (or a Glue crawler does it).

### Step 7 — Spectrum: hot and cold data in one query: `sql/02_spectrum.sql`

`CREATE EXTERNAL SCHEMA … FROM DATA CATALOG` maps a Glue database into Redshift; `IAM_ROLE default` uses the cluster's default role (read-only on the lake). The query compares the **local** fact table (hot, last 90 days) with the **S3** copy (cold, any age) month by month; the `WHERE year >= '2020'` filter on a partition column lets Spectrum skip folders entirely (**partition pruning**).

```sql
-- ═══════════════════════════════════════════════════════════════
-- Redshift Spectrum: Query 7 years of S3 data without loading
--
-- WHY Spectrum for historical data:
-- 50TB of 7-year history loaded into Redshift = $1,150/month storage
-- Spectrum: $5/TB scanned. A typical historical query scans ~100GB
-- of Parquet (partitioned + columnar) = $0.50 per query.
-- At 50 historical queries/day = $25/day = $750/month.
-- Spectrum wins at < $1,150/month usage.
-- ═══════════════════════════════════════════════════════════════

-- Create external schema pointing to the Glue Catalog database built in Step 3.2
-- (IAM_ROLE default = the cluster's default role, cloudmetrics-redshift-spectrum-role)
CREATE EXTERNAL SCHEMA IF NOT EXISTS spectrum_raw
FROM DATA CATALOG
DATABASE 'cloudmetrics_raw'
IAM_ROLE default;

-- Hot data from Redshift local tables next to history from S3 — side by side per month.
-- Spectrum handles the S3 read; Redshift handles the local data and the join.
-- For the months you loaded, redshift_count and spectrum_count should match.
SELECT
    COALESCE(r.year_num, s.year_num)    AS year,
    COALESCE(r.month_num, s.month_num)  AS month,
    r.event_count                       AS redshift_count,
    s.event_count                       AS spectrum_count
FROM (
    -- Hot data (last 90 days) — from Redshift local tables
    SELECT
        d.year_num,
        d.month_num,
        COUNT(*) AS event_count
    FROM analytics.fact_api_events f
    JOIN analytics.dim_date d ON f.event_date = d.full_date
    WHERE f.event_date >= CURRENT_DATE - 90
    GROUP BY d.year_num, d.month_num
) r
FULL OUTER JOIN (
    -- Historical data (up to 7 years) — from S3 via Spectrum
    SELECT
        EXTRACT(YEAR  FROM event_timestamp)::INT AS year_num,
        EXTRACT(MONTH FROM event_timestamp)::INT AS month_num,
        COUNT(*)                                  AS event_count
    FROM spectrum_raw.events
    WHERE year >= '2020'   -- Partition pruning! (narrow this to the years you need)
    GROUP BY 1, 2
) s ON r.year_num = s.year_num AND r.month_num = s.month_num
ORDER BY 1, 2;
```

```bash
python scripts/rs.py sql/02_spectrum.sql
```

**Expected output:**

```
-> CREATE EXTERNAL SCHEMA IF NOT EXISTS spectrum_raw
-> SELECT
year | month | redshift_count | spectrum_count
2026 | 3 | 10000 | 10000
```

✅ **Checkpoint:** both counts match — a **cross-system reconciliation**: what you loaded equals what is in the lake. You can now explain when to load (hot, frequently queried) and when to leave data in S3 for Spectrum (cold history, $5/TB scanned).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No files under s3://…/day=DD/` | Wrong date (default is *today*, UTC) | Pass your data day: `python scripts/08_load_day.py YYYY-MM-DD` |
| `Query failed: … Not authorized to get credentials of role` / `S3ServiceException: Access Denied` | Cluster default role missing, or role lacks S3/KMS read | Check `DefaultIamRoleArn` in `aws redshift describe-clusters`; re-run `python scripts/03_iam_roles.py` |
| `Query failed: … content_length` | Manifest entry without `meta.content_length` | Use the script; Parquet manifests need sizes |
| `Query failed: … columns … do not match` / `incompatible Parquet schema` | Staging column count/order/type differs from the Parquet file | Compare `sql/01_schema.sql` staging table with the Glue table (Lab 04 Step 3) |
| Spectrum count is 0 / `spectrum_count` empty | Partitions not registered | `python scripts/09_register_partitions.py` |
| `External schema … already exists` with a different database | Old schema from an earlier attempt | `DROP SCHEMA spectrum_raw;` then re-run the file |

## Teardown

**Mandatory today** — delete the cluster (Lab 07 does not need it; the guided lab builds a fresh one):

```bash
aws redshift delete-cluster --cluster-identifier "$CLUSTER_ID" --skip-final-cluster-snapshot --query 'Cluster.ClusterStatus'
aws redshift wait cluster-deleted --cluster-identifier "$CLUSTER_ID"
aws redshift delete-cluster-subnet-group --cluster-subnet-group-name cloudmetrics-subnet-group
SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=cloudmetrics-redshift-sg --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 delete-security-group --group-id "$SG_ID"
aws s3 rm "s3://$BUCKET/temp/lab06/" --recursive
aws redshift describe-clusters --query 'Clusters[].ClusterIdentifier'
```

**Expected output:** `"deleting"`, then `[]`. Keep the bucket, key, Glue table and roles until Lab 07's clean-slate teardown.

## Key takeaways

- Manifest = exact, auditable file list; Parquet manifests need `content_length`.
- COPY into staging, then delete + insert the day in **one** transaction → idempotent, all-or-nothing loads.
- Verify every load: counts, `sys_load_history`, `stl_load_errors`, and a reconciliation against the source.
- Spectrum reads S3 through the Glue Catalog — register partitions, filter on partition columns.

## Next

[Lab 07 — Cost Controls, Monitoring and Cost Analysis](Lab_07_Cost_Controls_Monitoring_and_Cost_Analysis.md)
