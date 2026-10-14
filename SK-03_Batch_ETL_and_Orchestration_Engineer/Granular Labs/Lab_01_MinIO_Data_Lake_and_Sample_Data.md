# Lab 01 — MinIO Data Lake and Sample Data

> **Goal:** Create UrbanGear's `urbangear-data-lake` bucket and land five days of raw JSON order files in it, exactly where the guided lab's pipeline expects them.
> **Time:** 90–120 min | **Difficulty:** Beginner | **Prerequisites:** Lab 00
> **Guided lab coverage:** SCENARIO; Part 1 Step 1.1 (`S3_RAW` path) and Step 1.3 (input `raw/orders/dt=<date>/`); Part 2 Step 2.2 (`check_source_data` lists keys) and Step 2.3 (sensor key `raw/orders/dt={{ ds }}/part-*.json`)

## Learning objectives

By the end of this lab you can:

- Explain buckets, keys, prefixes, and why "folders" in S3/MinIO are an illusion.
- Describe the data-lake zones the guided lab uses: `raw/`, `processed/`, `rejected/`, `metrics/`.
- Read and write objects with **boto3** (the AWS Python SDK) against MinIO.
- Explain **JSON Lines** and **Hive-style partitioning** (`dt=2026-07-01`).
- Generate reproducible test data with a fixed random seed.

## Steps

### Step 1 — Understand the data-lake layout

UrbanGear (the guided lab's retailer) lands one export of orders per day. Every zone is a **key prefix** inside **one bucket**, `urbangear-data-lake` — the same layout as the guided lab's `PipelineConfig`:

```
urbangear-data-lake/
├── raw/orders/dt=2026-07-01/part-00000.json        ← files exactly as received (this lab)
├── processed/orders/order_year=2026/order_month=7/order_day=1/part-*.parquet   (Lab 04)
├── rejected/orders/dt=2026-07-01/part-*.parquet    ← dead-letter: rows that failed checks (Lab 04)
└── metrics/orders/dt=2026-07-01/metrics.json       ← run statistics for the quality gate (Lab 04)
```

Three facts about object storage shape everything else:

1. **Keys are just strings.** `raw/orders/dt=2026-07-01/part-00000.json` *looks* like folders, but there is no folder object — the console draws one wherever it sees `/`.
2. **Objects are immutable.** You never edit part of an object; you replace it whole. Pipelines therefore *overwrite* a day's output instead of appending to it.
3. **Listing is a query by prefix.** "Give me every key starting with `raw/orders/dt=2026-07-01/`" is how a pipeline finds a day's input — which is why the date is *in* the key.

`dt=2026-07-01` is **Hive-style partitioning** (`column=value` in the path). Spark, Athena and Glue recognise it automatically.

**Expected output:** none — read and make sure you can explain the three facts to someone else.

✅ **Checkpoint:** you can say what lives in each of the four zones.

### Step 2 — Create the bucket with code

Create `C:\DataEngineering\sk03\jobs\setup_buckets.py`:

```python
"""Create UrbanGear's data-lake bucket. Safe to re-run (idempotent)."""
import os

import boto3
from botocore.exceptions import ClientError

s3 = boto3.client(
    "s3",
    endpoint_url=os.environ["MINIO_ENDPOINT"],          # http://minio:9000 inside Docker
    aws_access_key_id=os.environ["MINIO_ACCESS_KEY"],
    aws_secret_access_key=os.environ["MINIO_SECRET_KEY"],
)

# One bucket. Zones (raw/, processed/, rejected/, metrics/) are key prefixes inside it.
BUCKETS = ["urbangear-data-lake"]

for bucket in BUCKETS:
    try:
        s3.head_bucket(Bucket=bucket)          # cheap "does it exist?" check
        print(f"exists : {bucket}")
    except ClientError:
        s3.create_bucket(Bucket=bucket)
        print(f"created: {bucket}")
```

- `endpoint_url` points boto3 at MinIO instead of AWS. That one argument is the *only* difference from code that talks to real S3.
- Credentials come from **environment variables** set in `docker-compose.yml` — never typed into the script.
- `head_bucket` then `create_bucket` makes the script **idempotent**: running it twice gives the same end state, no crash.

Run it twice:

```powershell
docker exec -it airflow-scheduler python /opt/airflow/jobs/setup_buckets.py
docker exec -it airflow-scheduler python /opt/airflow/jobs/setup_buckets.py
```

**Expected output:**

```
created: urbangear-data-lake
exists : urbangear-data-lake
```

✅ **Checkpoint:** http://localhost:9001 → Object Browser shows `urbangear-data-lake`.

### Step 3 — Meet the raw order format (JSON Lines)

The guided lab's source system exports **JSON Lines**: one JSON object per line, one line per **order line item** (an order with three products produces three lines sharing one `order_id`). Each line has the 17 fields of the guided lab's `RAW_ORDER_SCHEMA`:

| Field | Example | Notes |
|---|---|---|
| `order_id` | `ORD-20260701-00000` | Repeats for each product in the order |
| `customer_id` | `CUST-0241` | Sometimes `null` (bad row) |
| `platform` | `website` | Valid: `website`, `mobile_app`, `marketplace`; sometimes messy (`" Website "`) or invalid (`"fax"`) |
| `order_date` | `2026-07-01T20:05:41` | ISO timestamp; sometimes `2099-...` (future = bad row) |
| `product_id`, `product_name`, `category` | `P100`, `Trail Backpack`, `Bags` | 6 products |
| `quantity` | `2` | Sometimes negative (bad row) |
| `unit_price`, `discount_pct`, `shipping_cost`, `tax_amount` | `89.99`, `10.0`, `11.65`, `14.4` | `discount_pct` is sometimes `null` |
| `payment_method` | `gift_card` | Lower case in the source; the ETL upper-cases it |
| `shipping_country`, `shipping_state`, `shipping_city` | `US`, `CA`, `Los Angeles` | Country sometimes lower case |
| `is_returned` | `false` | Boolean |

About 3% of lines are deliberately dirty, and some lines are **corrupt** — truncated, not valid JSON at all — exactly the problems the guided lab's job must catch.

**Expected output:** none.

✅ **Checkpoint:** you can name the five kinds of dirt: null customer, negative quantity, future date, invalid platform, corrupt JSON.

### Step 4 — Write the data generator

Create `C:\DataEngineering\sk03\jobs\generate_orders.py`:

```python
"""Generate one day of UrbanGear raw orders as JSON Lines and upload to MinIO.

Same 17-field schema as the SK-03 guided lab. Deterministic: the same date and
dirty rate always produce exactly the same file (the random seed is the date).

Usage:
    python generate_orders.py --process_date 2026-07-01
    python generate_orders.py --process_date 2026-07-05 --dirty_rate 0.12   # a "bad day"
"""
import argparse
import json
import os
import random
from datetime import datetime, timedelta

import boto3

BUCKET = "urbangear-data-lake"
ORDERS_PER_DAY = 1000            # each order has 1-3 lines -> ~2,000 lines/day

PLATFORMS = ["website", "mobile_app", "marketplace"]
PRODUCTS = [  # (product_id, product_name, category, unit_price)
    ("P100", "Trail Backpack", "Bags",      89.99),
    ("P200", "Camp Stove",     "Cooking",   45.50),
    ("P300", "Headlamp",       "Lighting",  19.99),
    ("P400", "Tent 2P",        "Shelter",  199.00),
    ("P500", "Sleeping Bag",   "Sleeping", 129.00),
    ("P600", "Water Filter",   "Hydration", 34.95),
]
PAYMENTS = ["card", "paypal", "apple_pay", "gift_card"]
LOCATIONS = [  # (country, state, city)
    ("US", "CA", "Los Angeles"), ("US", "NY", "New York"), ("US", "TX", "Austin"),
    ("CA", "ON", "Toronto"),     ("GB", "ENG", "London"),
]
DIRT_TYPES = ["null_customer", "negative_qty", "future_date", "bad_platform", "corrupt_json"]


def make_lines(process_date: str, dirty_rate: float) -> list[str]:
    rng = random.Random(f"{process_date}-{dirty_rate}")   # deterministic per (date, rate)
    day = datetime.strptime(process_date, "%Y-%m-%d")
    lines = []
    for i in range(ORDERS_PER_DAY):
        order_id = f"ORD-{process_date.replace('-', '')}-{i:05d}"
        customer_id = f"CUST-{rng.randint(1, 400):04d}"
        platform = rng.choice(PLATFORMS)
        if rng.random() < 0.10:                       # messy-but-valid casing/spacing
            platform = f" {platform.title()} "
        order_ts = day + timedelta(seconds=rng.randint(0, 86_399))
        country, state, city = rng.choice(LOCATIONS)
        payment = rng.choice(PAYMENTS)
        for product_id, name, category, price in rng.sample(PRODUCTS, rng.randint(1, 3)):
            qty = rng.randint(1, 4)
            rec = {
                "order_id": order_id,
                "customer_id": customer_id,
                "platform": platform,
                "order_date": order_ts.strftime("%Y-%m-%dT%H:%M:%S"),
                "product_id": product_id,
                "product_name": name,
                "category": category,
                "quantity": qty,
                "unit_price": price,
                "discount_pct": rng.choice([0.0, 0.0, 0.0, 5.0, 10.0, 15.0, None]),
                "shipping_cost": round(rng.uniform(0, 15), 2),
                "tax_amount": round(qty * price * 0.08, 2),
                "payment_method": payment,
                "shipping_country": country.lower() if rng.random() < 0.05 else country,
                "shipping_state": state,
                "shipping_city": city,
                "is_returned": rng.random() < 0.03,
            }
            line = json.dumps(rec)
            # --- inject realistic dirt into a small share of lines ---
            if rng.random() < dirty_rate:
                dirt = rng.choice(DIRT_TYPES)
                if dirt == "null_customer":
                    rec["customer_id"] = None
                elif dirt == "negative_qty":
                    rec["quantity"] = -rng.randint(1, 3)
                elif dirt == "future_date":
                    rec["order_date"] = "2099-01-01T00:00:00"
                elif dirt == "bad_platform":
                    rec["platform"] = "fax"
                line = json.dumps(rec)
                if dirt == "corrupt_json":
                    line = line[:-20]                 # truncated line = broken JSON
            lines.append(line)
    return lines


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--process_date", required=True, help="YYYY-MM-DD")
    parser.add_argument("--dirty_rate", type=float, default=0.03)
    args = parser.parse_args()

    lines = make_lines(args.process_date, args.dirty_rate)
    body = ("\n".join(lines) + "\n").encode("utf-8")

    s3 = boto3.client(
        "s3",
        endpoint_url=os.environ["MINIO_ENDPOINT"],
        aws_access_key_id=os.environ["MINIO_ACCESS_KEY"],
        aws_secret_access_key=os.environ["MINIO_SECRET_KEY"],
    )
    key = f"raw/orders/dt={args.process_date}/part-00000.json"
    s3.put_object(Bucket=BUCKET, Key=key, Body=body)
    print(f"uploaded s3://{BUCKET}/{key} ({len(lines)} lines, {len(body):,} bytes)")


if __name__ == "__main__":
    main()
```

- **`random.Random(f"{process_date}-{dirty_rate}")`** — a fixed seed per date: re-running a date reproduces the *identical* file. Reproducible data is how you test pipelines, and why the expected numbers in these labs match yours.
- **`--process_date`** — the same argument name the guided lab's Spark job uses.
- **`--dirty_rate`** — 3% by default; you'll make a deliberately "bad day" at 12% to trip the quality gate in Lab 07.
- **Key** `raw/orders/dt=<date>/part-00000.json` — matches the guided lab's sensor pattern `raw/orders/dt={{ ds }}/part-*.json`.

**Expected output:** none yet.

✅ **Checkpoint:** the file is saved in `jobs\`.

### Step 5 — Generate five days of orders

Four normal days, plus one bad day for Lab 07:

```powershell
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-01
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-02
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-03
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-04
docker exec -it airflow-scheduler python /opt/airflow/jobs/generate_orders.py --process_date 2026-07-05 --dirty_rate 0.12
```

**Expected output** (exact — the data is deterministic):

```
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-01/part-00000.json (1968 lines, 848,812 bytes)
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-02/part-00000.json (1989 lines, 858,532 bytes)
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-03/part-00000.json (2004 lines, 864,842 bytes)
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-04/part-00000.json (2000 lines, 863,405 bytes)
uploaded s3://urbangear-data-lake/raw/orders/dt=2026-07-05/part-00000.json (2004 lines, 863,182 bytes)
```

Run the first command again — same line count, same byte count, object replaced. That is idempotency again.

✅ **Checkpoint:** in the MinIO console, `urbangear-data-lake` → `raw/orders/` shows five `dt=` "folders", each with one `part-00000.json` of about 830–845 KiB.

### Step 6 — List and read objects like a pipeline does

The guided lab's `check_source_data` task lists a day's keys; its quality gate reads a small JSON object. Practise both with boto3. Open Python inside the container:

```powershell
docker exec -it airflow-scheduler python
```

```python
import os, json, boto3
s3 = boto3.client("s3", endpoint_url=os.environ["MINIO_ENDPOINT"],
                  aws_access_key_id=os.environ["MINIO_ACCESS_KEY"],
                  aws_secret_access_key=os.environ["MINIO_SECRET_KEY"])

# 1. Discover input by prefix — a paginator handles >1,000 keys correctly
pages = s3.get_paginator("list_objects_v2").paginate(
    Bucket="urbangear-data-lake", Prefix="raw/orders/")
for page in pages:
    for obj in page.get("Contents", []):
        print(obj["Key"], obj["Size"])

# 2. An empty prefix is NOT an error — "Contents" is simply missing
resp = s3.list_objects_v2(Bucket="urbangear-data-lake", Prefix="raw/orders/dt=2026-07-09/")
print("files for 07-09:", len(resp.get("Contents", [])))

# 3. Read one object and look at its first line
body = s3.get_object(Bucket="urbangear-data-lake",
                     Key="raw/orders/dt=2026-07-01/part-00000.json")["Body"].read().decode()
lines = body.splitlines()
print(json.dumps(json.loads(lines[0]), indent=2))

# 4. How many lines are not even valid JSON?
def is_json(line):
    try:
        json.loads(line); return True
    except json.JSONDecodeError:
        return False
print("corrupt lines:", sum(not is_json(l) for l in lines))
```

**Expected output:**

```
raw/orders/dt=2026-07-01/part-00000.json 848812
raw/orders/dt=2026-07-02/part-00000.json 858532
raw/orders/dt=2026-07-03/part-00000.json 864842
raw/orders/dt=2026-07-04/part-00000.json 863405
raw/orders/dt=2026-07-05/part-00000.json 863182
files for 07-09: 0
{
  "order_id": "ORD-20260701-00000",
  "customer_id": "CUST-0241",
  "platform": " Website ",
  "order_date": "2026-07-01T20:05:41",
  "product_id": "P100",
  "product_name": "Trail Backpack",
  "category": "Bags",
  "quantity": 2,
  "unit_price": 89.99,
  "discount_pct": 10.0,
  "shipping_cost": 11.65,
  "tax_amount": 14.4,
  "payment_method": "gift_card",
  "shipping_country": "US",
  "shipping_state": "CA",
  "shipping_city": "Los Angeles",
  "is_returned": false
}
corrupt lines: 15
```

Notice the very first record already has a messy platform (`" Website "`). Type `exit()` to leave Python.

✅ **Checkpoint:** five keys listed, `files for 07-09: 0`, and `corrupt lines: 15`.

### Step 7 — Peek behind the curtain (and why you shouldn't touch it)

```powershell
docker exec -it minio ls /bitnami/minio/data
docker compose logs --tail 5 minio
```

**Expected output:** the first command lists `urbangear-data-lake` (MinIO's private on-disk layout); the second shows MinIO's last log lines.

Never edit files under `/bitnami/minio/data` directly — it corrupts MinIO's metadata. Code always goes through the S3 API.

✅ **Checkpoint:** you know where the data physically lives, and that you only ever use the API.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `KeyError: 'MINIO_ENDPOINT'` | Script run on Windows, not in the container | Always `docker exec -it airflow-scheduler python /opt/airflow/jobs/...` |
| `EndpointConnectionError ... localhost:9000` | `localhost` used inside a container | Use `http://minio:9000` (it comes from `MINIO_ENDPOINT`) |
| `NoSuchBucket` | Step 2 not run, or a typo | Run `setup_buckets.py`; the name is `urbangear-data-lake` |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch` | Credentials changed in compose | Restore `minioadmin` / `minioadmin123`, `docker compose up -d` |
| `python: can't open file '/opt/airflow/jobs/...'` | File saved in the wrong folder | It must be in `C:\DataEngineering\sk03\jobs\` |
| Line/byte counts differ from Step 5 | Generator edited, or a different `--dirty_rate` | Copy the generator exactly; use the commands exactly |

## Key takeaways

- One bucket, `urbangear-data-lake`, with zone prefixes `raw/`, `processed/`, `rejected/`, `metrics/` — identical to the guided lab.
- The key contract is `raw/orders/dt=YYYY-MM-DD/part-*.json`. Every later tool (Spark, the Airflow sensor, `check_source_data`) depends on it.
- boto3 + `endpoint_url` = the same code for MinIO and AWS S3. Credentials come from the environment.
- Empty listings are normal (`resp.get("Contents", [])`); more than 1,000 keys needs a paginator.
- Deterministic test data (fixed seeds) makes every expected output reproducible.

## Next

[Lab 02 — PySpark Sessions, Schemas and JSON](Lab_02_PySpark_Sessions_Schemas_and_JSON.md): start writing `etl_job.py` — the guided lab's `PipelineConfig`, explicit schema and tuned SparkSession — and read these JSON files with Spark.
