# Lab 03 — Kinesis Data Streams and a Resilient Producer

> **Goal:** Size, create and use the `cloudmetrics-events` Kinesis stream, and write a producer that never silently drops records when `PutRecords` partially fails.
> **Time:** 90–120 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 02
> **Cost:** ~$0.10 — an ON_DEMAND stream bills **~$0.04 per hour while it exists** plus $0.08/GB written. **Delete the stream in Teardown before you stop for the day.**
> **Guided lab coverage:** PART 3 — Step 3.1 (create the stream, ON_DEMAND vs PROVISIONED shard math) and Step 3.4 (resilient producer with `PutRecords`)

## Learning objectives

By the end of this lab you can:

- Explain shards, partition keys, hot shards and retention, and calculate shard counts from throughput.
- Choose ON_DEMAND vs PROVISIONED capacity with a cost argument.
- Send records with `PutRecord` (CLI) and `PutRecords` (boto3), and read them back.
- Handle `PutRecords` partial failures with retries and exponential backoff — and **test** that logic offline with botocore's `Stubber`.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Kinesis concepts and shard math

| Concept | Meaning |
|---|---|
| **Stream** | An ordered, replayable log of records, kept for 24 hours by default |
| **Shard** | A unit of capacity: write **1 MB/s or 1,000 records/s** (whichever comes first), read 2 MB/s |
| **Partition key** | A string you choose per record; its hash decides the shard. Same key → same shard → ordered |
| **Hot shard** | One key (e.g. one huge customer) sends more than a shard can take → throttling (`ProvisionedThroughputExceededException`) |
| **PROVISIONED** | You choose the shard count; $0.015 per shard-hour; you must re-shard before peaks |
| **ON_DEMAND** | AWS scales shards for you (starts at 4); ~$0.04 per stream-hour + $0.08/GB written |

Save and run the capacity calculation for CloudMetrics:

```python
# scripts/lab03_shard_math.py — Kinesis capacity planning for CloudMetrics (no AWS calls)
import math

EVENT_BYTES = 1_000                      # ~1 KB per API event
SHARD_MB_PER_SEC = 1.0                   # write limit per shard: 1 MB/s ...
SHARD_RECORDS_PER_SEC = 1_000            # ... or 1,000 records/s, whichever is hit first
SHARD_HOUR_USD = 0.015                   # PROVISIONED shard-hour price (us-east-1)
HOURS_PER_MONTH = 720                    # 30-day month

for label, events_per_sec in (("average", 2_315), ("peak", 23_150)):
    mb_per_sec = events_per_sec * EVENT_BYTES / 1_000_000
    shards = max(math.ceil(mb_per_sec / SHARD_MB_PER_SEC),
                 math.ceil(events_per_sec / SHARD_RECORDS_PER_SEC))
    monthly = shards * SHARD_HOUR_USD * HOURS_PER_MONTH
    print(f"{label:7}: {events_per_sec:>6,} events/s = {mb_per_sec:6.3f} MB/s -> "
          f"{shards:>2} shards -> ${monthly:,.0f}/month in PROVISIONED mode")
```

```bash
python scripts/lab03_shard_math.py
```

**Expected output:**

```
average:  2,315 events/s =  2.315 MB/s ->  3 shards -> $32/month in PROVISIONED mode
peak   : 23,150 events/s = 23.150 MB/s -> 24 shards -> $259/month in PROVISIONED mode
```

✅ **Checkpoint:** you can reproduce the guided lab's comment "PROVISIONED would require 24 shards … $259/month" and argue why CloudMetrics picks ON_DEMAND anyway: a 10x daily spike means either paying for 24 shards all day or re-sharding twice a day.

### Step 2 — Create the stream: `scripts/04_create_stream.py`

A **waiter** polls an API until a condition is true (here: the stream is `ACTIVE`), so your script does not race ahead.

```python
# scripts/04_create_stream.py
import boto3

from cm_config import REGION, STREAM_NAME, TAGS

kinesis = boto3.client("kinesis", region_name=REGION)

# Create the stream in ON_DEMAND mode
# WHY ON_DEMAND for CloudMetrics:
#   Average: 2,315 events/sec (~2.3MB/sec)
#   Peak:    23,150 events/sec (~23MB/sec)
#   ON_DEMAND scales to 200MB/sec write automatically
#   PROVISIONED would require 24 shards (23MB ÷ 1MB/shard)
#     at $0.015/shard-hour = $259/month — comparable to ON_DEMAND
#     but requires manual scaling

try:
    kinesis.create_stream(
        StreamName=STREAM_NAME,
        StreamModeDetails={"StreamMode": "ON_DEMAND"},
        Tags=TAGS,
    )
except kinesis.exceptions.ResourceInUseException:
    print(f"Stream {STREAM_NAME} already exists — reusing it")

# Wait for the stream to become ACTIVE
waiter = kinesis.get_waiter("stream_exists")
waiter.wait(StreamName=STREAM_NAME)
print(f"Stream {STREAM_NAME} is ACTIVE")
```

```bash
python scripts/04_create_stream.py
aws kinesis describe-stream-summary --stream-name "$STREAM_NAME" \
    --query 'StreamDescriptionSummary.{Status:StreamStatus,Mode:StreamModeDetails.StreamMode,OpenShards:OpenShardCount,RetentionHours:RetentionPeriodHours}'
```

**Expected output:**

```
Stream cloudmetrics-events is ACTIVE
{
    "Status": "ACTIVE",
    "Mode": "ON_DEMAND",
    "OpenShards": 4,
    "RetentionHours": 24
}
```

✅ **Checkpoint:** the stream is ACTIVE with 4 shards. **The clock is now running (~$0.04/hour)** — finish this lab in one sitting or delete the stream (Teardown) and recreate it later with the same script.

### Step 3 — Put one record with the CLI

AWS CLI v2 expects binary `--data` to be base64. `--cli-binary-format raw-in-base64-out` lets you pass plain text:

```bash
aws kinesis put-record \
    --stream-name "$STREAM_NAME" \
    --partition-key cust-0001 \
    --cli-binary-format raw-in-base64-out \
    --data '{"event_id":"evt-cli-0001","customer_id":"cust-0001","event_type":"api_call"}'
```

**Expected output:**

```json
{
    "ShardId": "shardId-000000000002",
    "SequenceNumber": "49650000000000000000000000000000000000000000000000000002"
}
```

✅ **Checkpoint:** run it again with the same partition key — the `ShardId` is identical (same key → same shard → ordering preserved for that customer).

### Step 4 — The resilient producer: `scripts/06_producer.py`

`PutRecords` sends up to 500 records per call, but it can **partially fail**: the HTTP call succeeds, `FailedRecordCount` is > 0, and the failed entries carry an `ErrorCode`. A naive producer that only checks for exceptions silently loses data. The guided lab's producer retries **only the failed records**, with exponential backoff:

```python
# scripts/06_producer.py
import json
import logging
import time
from datetime import datetime, timezone

import boto3

from cm_config import REGION, STREAM_NAME

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
kinesis = boto3.client("kinesis", region_name=REGION)
BATCH_SIZE = 500   # Kinesis PutRecords maximum


def send_events(events: list, max_retries: int = 3) -> dict:
    """
    Send a batch of events to Kinesis using PutRecords.
    Handles partial failures with exponential backoff.

    Returns a summary dict with success/failure counts.
    """
    records = [
        {
            "Data": json.dumps(e).encode("utf-8"),
            # WHY partition by customer_id:
            # - Orders events from the same customer within a shard
            # - Distributes load evenly across shards (5K customers)
            # - Enables per-customer ordered processing
            "PartitionKey": e["customer_id"],
        }
        for e in events
    ]

    pending = records
    attempt = 0
    total_sent = 0

    while pending and attempt < max_retries:
        response = kinesis.put_records(
            StreamName=STREAM_NAME,
            Records=pending,
        )

        failed_count = response.get("FailedRecordCount", 0)

        if failed_count == 0:
            total_sent += len(pending)
            pending = []
            break

        # Collect failed records for retry
        # WHY: PutRecords returns the result per record in order.
        # Records with an ErrorCode field failed.
        retry_records = []
        first_error = None
        for i, result in enumerate(response["Records"]):
            if "ErrorCode" in result:
                retry_records.append(pending[i])
                first_error = first_error or result["ErrorCode"]
            else:
                total_sent += 1

        logger.warning(
            f"Attempt {attempt + 1}: {failed_count} records failed "
            f"({first_error} for first failure). Retrying..."
        )

        pending = retry_records
        attempt += 1
        # Exponential backoff: 2s, 4s, 8s
        time.sleep(2 ** attempt)

    total_failed = len(pending)
    if total_failed > 0:
        logger.error(f"{total_failed} records could not be delivered after {max_retries} retries")

    return {"sent": total_sent, "failed": total_failed}


def send_all_events(event_list: list) -> dict:
    """
    Split a large list into batches of 500 and send each batch.
    """
    results = {"sent": 0, "failed": 0}
    for i in range(0, len(event_list), BATCH_SIZE):
        batch = event_list[i:i + BATCH_SIZE]
        result = send_events(batch)
        results["sent"]   += result["sent"]
        results["failed"] += result["failed"]
    return results


if __name__ == "__main__":
    # Example usage — field names match the Glue table in Step 3.2
    now = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%fZ")   # Firehose-parsable ISO-8601
    sample_events = [
        {
            "event_id":            f"evt-{i:08d}",
            "customer_id":         f"cust-{i % 5000:04d}",
            "event_type":          "api_call",
            "endpoint":            "/v2/analytics/query",
            "method":              "POST",
            "status_code":         200,
            "response_time_ms":    145,
            "request_size_bytes":  2048,
            "response_size_bytes": 15360,
            "event_timestamp":     now,
            "region":              "us-east-1",
        }
        for i in range(10_000)
    ]

    summary = send_all_events(sample_events)
    print(f"Sent: {summary['sent']:,} | Failed: {summary['failed']:,}")
```

Read it before running it:

- `PartitionKey: e["customer_id"]` — one customer's events stay ordered on one shard.
- `response["Records"]` is in the **same order** as the records sent, so `pending[i]` is the record behind result `i`.
- `time.sleep(2 ** attempt)` waits 2 s, 4 s, 8 s between retries to let a throttled shard recover.
- `if __name__ == "__main__":` means importing the file (as the test in Step 5 does) sends nothing.

**Expected output:** none yet — you will test before you run.

✅ **Checkpoint:** you can explain what happens to record #2 of 3 if it is throttled once.

### Step 5 — Test the retry logic offline with `Stubber`

You cannot make Kinesis throttle on demand, so fake it: botocore's **Stubber** plugs canned responses into a real boto3 client. No AWS call is made and nothing is billed. Save `scripts/lab03_test_partial_failure.py`:

```python
# scripts/lab03_test_partial_failure.py — offline test of the producer's retry logic (no AWS calls)
import importlib
import json

from botocore.stub import Stubber

producer = importlib.import_module("06_producer")     # file names starting with a digit need importlib
producer.time.sleep = lambda seconds: None             # skip the real backoff during the test

events = [{"event_id": f"evt-{i}", "customer_id": f"cust-{i}"} for i in range(3)]

with Stubber(producer.kinesis) as stubber:
    # Call 1: record #2 is throttled, records #1 and #3 succeed
    stubber.add_response("put_records", {
        "FailedRecordCount": 1,
        "Records": [
            {"SequenceNumber": "1", "ShardId": "shardId-000000000000"},
            {"ErrorCode": "ProvisionedThroughputExceededException", "ErrorMessage": "Rate exceeded"},
            {"SequenceNumber": "3", "ShardId": "shardId-000000000001"},
        ],
    })
    # Call 2 must contain ONLY record #2 — and succeeds
    # (FailedRecordCount is left out: botocore's Stubber rejects 0 for it, and the producer reads it with .get(..., 0))
    stubber.add_response(
        "put_records",
        {"Records": [{"SequenceNumber": "4", "ShardId": "shardId-000000000002"}]},
        expected_params={"StreamName": producer.STREAM_NAME, "Records": [
            {"Data": json.dumps(events[1]).encode("utf-8"), "PartitionKey": "cust-1"}]},
    )
    result = producer.send_events(events)
    stubber.assert_no_pending_responses()

assert result == {"sent": 3, "failed": 0}, result
print("PASS: partial failure retried, nothing lost:", result)
```

```bash
python scripts/lab03_test_partial_failure.py
```

**Expected output:**

```
WARNING:06_producer:Attempt 1: 1 records failed (ProvisionedThroughputExceededException for first failure). Retrying...
PASS: partial failure retried, nothing lost: {'sent': 3, 'failed': 0}
```

✅ **Checkpoint:** the test passes. Break the producer on purpose (e.g. change `retry_records.append(pending[i])` to `pass`) and watch the test fail — then undo the change.

### Step 6 — Send 10,000 events and read them back

```bash
python scripts/06_producer.py
```

**Expected output:**

```
Sent: 10,000 | Failed: 0
```

Now read everything in the stream from the beginning (`TRIM_HORIZON`) of every shard. Save `scripts/lab03_read_stream.py`:

```python
# scripts/lab03_read_stream.py — count the records in every shard (reads from the beginning)
import json
from collections import Counter

import boto3

from cm_config import REGION, STREAM_NAME

kinesis = boto3.client("kinesis", region_name=REGION)
per_shard = Counter()
first_record = None

for shard in kinesis.list_shards(StreamName=STREAM_NAME)["Shards"]:
    iterator = kinesis.get_shard_iterator(StreamName=STREAM_NAME, ShardId=shard["ShardId"],
                                          ShardIteratorType="TRIM_HORIZON")["ShardIterator"]
    while iterator:
        response = kinesis.get_records(ShardIterator=iterator, Limit=10_000)
        for record in response["Records"]:
            per_shard[shard["ShardId"]] += 1
            first_record = first_record or json.loads(record["Data"])
        if response["MillisBehindLatest"] == 0:     # reached the newest record in this shard
            break
        iterator = response.get("NextShardIterator")

for shard_id, count in sorted(per_shard.items()):
    print(f"{shard_id}: {count:,}")
print(f"total: {sum(per_shard.values()):,}")
print("first record:", first_record)
```

```bash
python scripts/lab03_read_stream.py
```

**Expected output** (per-shard numbers vary; the total does not):

```
shardId-000000000000: 2,510
shardId-000000000001: 2,476
shardId-000000000002: 2,522
shardId-000000000003: 2,495
total: 10,002
first record: {'event_id': 'evt-...', 'customer_id': 'cust-...', ...}
```

(10,000 from the producer + the CLI record(s) from Step 3.)

✅ **Checkpoint:** the four shards hold roughly equal counts — 5,000 distinct customer IDs spread evenly. If one customer sent 10x more than everyone else, its shard would be "hot" (guided lab Reflection question 1).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ResourceNotFoundException` | Stream deleted, still creating, or wrong region | `python scripts/04_create_stream.py`; check `aws configure get region` |
| `Invalid base64` from `put-record` | Missing `--cli-binary-format raw-in-base64-out` | Add the flag |
| `ProvisionedThroughputExceededException` | Hot shard / too many writes per shard | The producer retries; long-term: better partition key or more shards |
| `ModuleNotFoundError: No module named '06_producer'` | Test run from the wrong folder | Run `python scripts/lab03_test_partial_failure.py` from `/c/DataEngineering/sk04` |
| `read_stream` total is 0 | Records older than 24 h expired, or the stream was recreated | Re-run the producer |

## Teardown

**Mandatory today** — the stream bills every hour it exists:

```bash
aws kinesis delete-stream --stream-name "$STREAM_NAME"
aws kinesis list-streams --query 'StreamNames'
```

**Expected output** (after ~30 seconds): `[]`. Lab 04 recreates the stream with `python scripts/04_create_stream.py`. Keep the bucket and key from Lab 02.

## Key takeaways

- Capacity = shards; the partition key decides the shard and the ordering.
- ON_DEMAND trades a per-hour stream fee for zero capacity management — right for spiky traffic.
- `PutRecords` can succeed *and* lose records: always inspect `FailedRecordCount` and retry only the failures with backoff.
- Test failure handling offline with `Stubber`; delete hourly-billed resources at the end of every session.

## Next

[Lab 04 — Glue Data Catalog, CloudFormation and Firehose](Lab_04_Glue_Catalog_CloudFormation_and_Firehose.md)
