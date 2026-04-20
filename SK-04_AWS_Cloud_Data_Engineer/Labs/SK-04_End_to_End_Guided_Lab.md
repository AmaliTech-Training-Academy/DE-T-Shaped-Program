# SK-04 END-TO-END GUIDED LAB: Building the CloudMetrics AWS Data Platform

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **cloud data engineer** brought in to migrate **CloudMetrics**, a B2B SaaS company, off their collapsing data infrastructure. CloudMetrics has 5,000 enterprise customers generating 200 million API events per day. All of it lands in a single PostgreSQL RDS instance (2TB) that is running out of headroom — fast.

The situation has become a business risk:

- **60 days to storage capacity**: The 2TB RDS instance will hit its limit in two months. Scaling it to handle 50TB would cost ~$15K/month — triple the current bill
- **Performance conflict**: Complex analytics queries block point-of-sale API requests, causing customer-facing timeouts during peak hours. Two enterprise customers complained last week
- **Compliance deadline**: GDPR and SOC2 require 7 years of event retention. RDS retains only 90 days
- **Product roadmap blocked**: Real-time customer health dashboards and anomaly detection are impossible on the current architecture

**Your job:** Design and build a complete AWS data platform — S3 data lake, Kinesis ingestion, Redshift analytics warehouse — with proper IAM security, cost controls, and lifecycle management.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why is this the right migration pattern?** | Moving analytics off the operational database is the most common cloud data engineering engagement. The pattern — RDS for transactions, S3/Kinesis for ingestion, Redshift for analytics — appears across retail, fintech, healthcare, and SaaS clients. |
| **Why Kinesis instead of direct S3 writes?** | 200M events/day = ~2,315 events/second average with 10x peaks at 23K/sec. Direct S3 writes from application code cannot batch efficiently at this rate and create millions of tiny files. Kinesis absorbs the throughput spike and Firehose batches it into 128MB Parquet files. |
| **When would you recommend Athena instead of Redshift?** | If the query pattern is ad-hoc and infrequent, Athena ($5/TB scanned) is cheaper than a provisioned Redshift cluster. When query volume is high and predictable, Redshift's per-cluster cost amortizes. This lab uses Redshift because CloudMetrics has 20 analysts running queries continuously. |
| **Why does the cost math matter?** | The CTO needs a cost justification before approving the migration. Showing the architecture is $4K/month vs $3.2K/month (current) AND unlimited scale vs 2TB cap is how you get sign-off. |

---

## WHAT YOU WILL BUILD

```
CloudMetrics AWS Data Platform:
│
├── 1. S3 DATA LAKE
│   ├── raw/          (JSON events from Kinesis, KMS encrypted)
│   ├── processed/    (Parquet, partitioned by year/month/day)
│   ├── curated/      (star schema for Redshift Spectrum)
│   ├── rejected/     (quality failures + Firehose errors)
│   └── temp/         (ETL working area, expires in 7 days)
│
├── 2. KINESIS PIPELINE
│   ├── cloudmetrics-events (Data Stream, ON_DEMAND)
│   └── cloudmetrics-events-to-s3 (Firehose → JSON → Parquet)
│
├── 3. REDSHIFT DATA WAREHOUSE
│   ├── analytics.fact_api_events (dist: customer_id, sort: timestamp)
│   ├── analytics.dim_customer    (dist: ALL — 5K rows)
│   ├── analytics.dim_date        (dist: ALL)
│   └── spectrum_raw (external schema over S3 for 7-year history)
│
├── 4. IAM SECURITY MODEL
│   ├── cloudmetrics-kinesis-producer-role
│   ├── cloudmetrics-firehose-delivery-role
│   ├── cloudmetrics-glue-etl-role
│   └── cloudmetrics-redshift-spectrum-role
│
└── 5. COST CONTROLS
    ├── S3 lifecycle policy (Standard → IA → Glacier → Deep Archive)
    ├── AWS Budget alert ($5,000/month)
    └── CloudWatch alarms (Kinesis age, Redshift CPU, S3 storage)
```

### Time Estimate: 8–10 hours

---

## PREREQUISITES

```bash
# AWS CLI installed and configured with a non-root IAM user
aws configure
# Verify: aws sts get-caller-identity

# Python packages
pip install boto3 pandas

# Access to an AWS account (sandbox or personal)
# Optional: LocalStack for fully local testing
```

---

## PART 1: S3 DATA LAKE SETUP (60 min)

### Step 1.1: Create and Configure the Data Lake Bucket

**WHY:** A bucket with no security configuration is a security incident waiting to happen. Enabling versioning, blocking public access, and applying KMS encryption should happen at creation time — retrofitting these later is operationally risky.

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════════
# CloudMetrics S3 Data Lake — Initial Setup
# Run once per environment (dev / staging / prod)
# ═══════════════════════════════════════════════════════════════

BUCKET="cloudmetrics-data-lake-prod"
REGION="us-east-1"
KMS_ALIAS="alias/cloudmetrics-data-key"

# 1. Create the bucket
aws s3api create-bucket \
    --bucket "$BUCKET" \
    --region "$REGION"

# 2. Block ALL public access — no exceptions
aws s3api put-public-access-block \
    --bucket "$BUCKET" \
    --public-access-block-configuration \
        "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# 3. Enable versioning
# WHY: Protects against accidental overwrites and deletes.
# Required for compliance (7-year retention with point-in-time recovery).
aws s3api put-bucket-versioning \
    --bucket "$BUCKET" \
    --versioning-configuration Status=Enabled

# 4. Create a Customer Managed Key (CMK) for encryption
# WHY CMK over SSE-S3: CMK gives us key rotation control,
# CloudTrail audit of every decrypt operation, and cross-account
# access management. $1/month per key.
aws kms create-key \
    --description "CloudMetrics data lake encryption key" \
    --region "$REGION"

aws kms create-alias \
    --alias-name "$KMS_ALIAS" \
    --target-key-id "$(aws kms describe-key --key-id "$KMS_ALIAS" \
                        --query 'KeyMetadata.KeyId' --output text 2>/dev/null \
                        || aws kms list-aliases --query \
                        "Aliases[?AliasName=='$KMS_ALIAS'].TargetKeyId" \
                        --output text)"

# 5. Enable default KMS encryption
aws s3api put-bucket-encryption \
    --bucket "$BUCKET" \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "'"$KMS_ALIAS"'"
            },
            "BucketKeyEnabled": true
        }]
    }'

echo "S3 bucket $BUCKET configured successfully"
```

### Step 1.2: Create Lifecycle Policy

**WHY:** Without a lifecycle policy, every byte of raw JSON stays in S3 Standard ($23/TB/month) forever. With the policy below, data automatically migrates to cheaper tiers as it ages — raw data that was accessed daily in week 1 costs $1/TB/month after 1 year.

```python
import boto3
import json

s3 = boto3.client("s3", region_name="us-east-1")

lifecycle_policy = {
    "Rules": [
        {
            "ID": "RawZone-CostTiering",
            "Filter": {"Prefix": "raw/"},
            "Status": "Enabled",
            "Transitions": [
                {"Days": 30,  "StorageClass": "STANDARD_IA"},     # $12.50/TB
                {"Days": 90,  "StorageClass": "GLACIER_IR"},       # $4.00/TB
                {"Days": 365, "StorageClass": "DEEP_ARCHIVE"},     # $0.99/TB
            ],
            # Compliance: 7 years retention, then delete
            "Expiration": {"Days": 2555},   # 7 years
        },
        {
            "ID": "ProcessedZone-CostTiering",
            "Filter": {"Prefix": "processed/"},
            "Status": "Enabled",
            "Transitions": [
                {"Days": 90,  "StorageClass": "STANDARD_IA"},
                {"Days": 365, "StorageClass": "GLACIER_IR"},
            ],
        },
        {
            "ID": "TempZone-AutoDelete",
            "Filter": {"Prefix": "temp/"},
            "Status": "Enabled",
            "Expiration": {"Days": 7},
        },
        {
            "ID": "CleanupIncompleteMultipartUploads",
            "Filter": {"Prefix": ""},
            "Status": "Enabled",
            # Delete orphaned multipart uploads after 7 days.
            # WHY: Incomplete MPUs accumulate silently and
            # incur storage charges with no corresponding object.
            "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7},
        },
    ]
}

s3.put_bucket_lifecycle_configuration(
    Bucket="cloudmetrics-data-lake-prod",
    LifecycleConfiguration=lifecycle_policy,
)
print("Lifecycle policy applied")
```

---

## PART 2: KINESIS INGESTION PIPELINE (60 min)

### Step 2.1: Create the Data Stream

**WHY:** ON\_DEMAND mode auto-scales shards based on actual traffic. CloudMetrics has variable traffic (10x spike during US business hours). The $0.08/GB cost is predictable and eliminates the 3 AM "add shards before the morning peak" operational burden.

```python
import boto3

kinesis = boto3.client("kinesis", region_name="us-east-1")

# Create the stream in ON_DEMAND mode
# WHY ON_DEMAND for CloudMetrics:
#   Average: 2,315 events/sec (~2.3MB/sec)
#   Peak:    23,150 events/sec (~23MB/sec)
#   ON_DEMAND scales to 200MB/sec write automatically
#   PROVISIONED would require 24 shards (23MB ÷ 1MB/shard)
#     at $0.015/shard-hour = $259/month — comparable to ON_DEMAND
#     but requires manual scaling

response = kinesis.create_stream(
    StreamName="cloudmetrics-events",
    StreamModeDetails={"StreamMode": "ON_DEMAND"},
)

# Wait for the stream to become ACTIVE
waiter = kinesis.get_waiter("stream_exists")
waiter.wait(StreamName="cloudmetrics-events")
print("Stream cloudmetrics-events is ACTIVE")
```

### Step 2.2: Resilient Event Producer

**WHY:** PutRecords is the right API for batch sends — one call for up to 500 records instead of 500 calls. But PutRecords has partial failures: some records succeed, some fail. The code below handles partial failures with exponential backoff rather than silently dropping records.

```python
import boto3
import json
import time
import logging
from datetime import datetime

logger = logging.getLogger(__name__)
kinesis = boto3.client("kinesis", region_name="us-east-1")
STREAM_NAME = "cloudmetrics-events"
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
    total_failed = 0

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
        for i, result in enumerate(response["Records"]):
            if "ErrorCode" in result:
                retry_records.append(pending[i])
            else:
                total_sent += 1

        logger.warning(
            f"Attempt {attempt + 1}: {failed_count} records failed "
            f"({result['ErrorCode']} for first failure). Retrying..."
        )

        pending = retry_records
        attempt += 1
        # Exponential backoff: 1s, 2s, 4s
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


# Example usage
sample_events = [
    {
        "event_id":           f"evt-{i:08d}",
        "customer_id":        f"cust-{i % 5000:04d}",
        "event_type":         "api_call",
        "endpoint":           "/v2/analytics/query",
        "method":             "POST",
        "status_code":        200,
        "response_time_ms":   145,
        "request_size_bytes": 2048,
        "response_size_bytes": 15360,
        "timestamp":          datetime.utcnow().isoformat() + "Z",
        "region":             "us-east-1",
    }
    for i in range(10_000)
]

summary = send_all_events(sample_events)
print(f"Sent: {summary['sent']:,} | Failed: {summary['failed']:,}")
```

### Step 2.3: Firehose Delivery Stream (CloudFormation)

**WHY:** Firehose is defined as CloudFormation rather than boto3 because it has many interdependent configuration blocks. CloudFormation validates the full configuration as a unit and enables version-controlled re-deployment.

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "CloudMetrics Kinesis Firehose → S3 with Parquet conversion",

    "Resources": {
        "EventsFirehose": {
            "Type": "AWS::KinesisFirehose::DeliveryStream",
            "Properties": {
                "DeliveryStreamName": "cloudmetrics-events-to-s3",
                "DeliveryStreamType": "KinesisStreamAsSource",
                "KinesisStreamSourceConfiguration": {
                    "KinesisStreamARN": {
                        "Fn::Sub": "arn:aws:kinesis:${AWS::Region}:${AWS::AccountId}:stream/cloudmetrics-events"
                    },
                    "RoleARN": {
                        "Fn::GetAtt": ["FirehoseKinesisRole", "Arn"]
                    }
                },
                "ExtendedS3DestinationConfiguration": {
                    "BucketARN": {
                        "Fn::Sub": "arn:aws:s3:::cloudmetrics-data-lake-prod"
                    },
                    "Prefix": "raw/events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
                    "ErrorOutputPrefix": "rejected/firehose/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
                    "BufferingHints": {
                        "SizeInMBs": 128,
                        "IntervalInSeconds": 300
                    },
                    "RoleARN": {
                        "Fn::GetAtt": ["FirehoseS3Role", "Arn"]
                    },
                    "DataFormatConversionConfiguration": {
                        "Enabled": true,
                        "InputFormatConfiguration": {
                            "Deserializer": {
                                "OpenXJsonSerDe": {}
                            }
                        },
                        "OutputFormatConfiguration": {
                            "Serializer": {
                                "ParquetSerDe": {
                                    "Compression": "SNAPPY"
                                }
                            }
                        },
                        "SchemaConfiguration": {
                            "DatabaseName": "cloudmetrics_raw",
                            "TableName":    "events",
                            "Region":       {"Ref": "AWS::Region"},
                            "RoleARN":      {"Fn::GetAtt": ["FirehoseGlueRole", "Arn"]}
                        }
                    }
                }
            }
        }
    }
}
```

---

## PART 3: REDSHIFT DATA WAREHOUSE (90 min)

### Step 3.1: Create the Cluster

```python
import boto3

redshift = boto3.client("redshift", region_name="us-east-1")

# Create a 3-node ra3.xlplus cluster
# WHY ra3.xlplus x3:
#   - Managed storage: data in S3 RMS (unlimited), hot cache in local NVMe
#   - 3 nodes = 6 slices (2 slices per ra3.xlplus)
#   - Good starting point: add nodes when query queue builds up
#   - On-Demand: $0.482/node-hour → 3 nodes × $0.482 × 730hr = ~$1,056/month
#   - 1-year RI:  $0.302/node-hour → ~$662/month (37% savings)

response = redshift.create_cluster(
    ClusterIdentifier="cloudmetrics-prod",
    NodeType="ra3.xlplus",
    NumberOfNodes=3,
    DBName="analytics",
    MasterUsername="admin",
    MasterUserPassword="CHANGE_ME_USE_SECRETS_MANAGER",
    ClusterSubnetGroupName="private-subnet-group",
    VpcSecurityGroupIds=["sg-0abc123"],
    IamRoles=[
        "arn:aws:iam::123456789012:role/cloudmetrics-redshift-spectrum-role"
    ],
    Encrypted=True,
    KmsKeyId="alias/cloudmetrics-data-key",
    Tags=[
        {"Key": "Environment", "Value": "production"},
        {"Key": "CostCenter",  "Value": "data-platform"},
    ],
)
print(f"Cluster creation initiated: {response['Cluster']['ClusterStatus']}")
```

### Step 3.2: Table Design

**WHY each design choice is documented in comments — distribution and sort key decisions are the most consequential performance choices in Redshift and should be version-controlled alongside the DDL.**

```sql
-- ═══════════════════════════════════════════════════════════════
-- CloudMetrics Redshift Schema
-- ═══════════════════════════════════════════════════════════════

CREATE SCHEMA IF NOT EXISTS analytics;
CREATE SCHEMA IF NOT EXISTS staging;

-- ── FACT: API Events ─────────────────────────────────────────
-- Volume: 200M events/day → 6B rows/month
-- DISTKEY(customer_id): Most queries filter/group by customer.
--   5,000 unique customers = even distribution across 6 slices
--   (~1,000 customers per slice). JOIN with dim_customer is local.
-- SORTKEY(event_timestamp): >80% of queries have a date range filter.
--   Zone maps allow Redshift to skip entire 1MB blocks outside the range.
-- AZ64 encoding for all numeric/timestamp columns (AWS default, good general purpose).
-- BYTEDICT for method/event_type (< 20 unique values = dictionary perfect fit).
-- ZSTD for customer_id/endpoint (variable length strings, high compression).

CREATE TABLE analytics.fact_api_events (
    event_id              VARCHAR(36)    NOT NULL ENCODE ZSTD,
    customer_id           VARCHAR(36)    NOT NULL ENCODE ZSTD,
    event_type            VARCHAR(50)    NOT NULL ENCODE ZSTD,
    endpoint              VARCHAR(500)            ENCODE ZSTD,
    method                VARCHAR(10)             ENCODE BYTEDICT,
    status_code           SMALLINT                ENCODE AZ64,
    response_time_ms      INTEGER                 ENCODE AZ64,
    request_size_bytes    BIGINT                  ENCODE AZ64,
    response_size_bytes   BIGINT                  ENCODE AZ64,
    event_timestamp       TIMESTAMP      NOT NULL ENCODE AZ64,
    region                VARCHAR(20)             ENCODE BYTEDICT,
    event_date            DATE           NOT NULL ENCODE AZ64,
    etl_loaded_at         TIMESTAMP               DEFAULT GETDATE()
)
DISTSTYLE KEY
DISTKEY (customer_id)
COMPOUND SORTKEY (event_timestamp);

-- ── DIM: Customer ─────────────────────────────────────────────
-- Volume: 5,000 rows — tiny
-- DISTSTYLE ALL: replicate the full 5K-row table to every node.
-- WHY: dim_customer is joined in almost every query.
--   With ALL distribution, the JOIN is always local — no network shuffle.
-- At 5K rows × ~500 bytes = 2.5MB total. ALL copies = 6 × 2.5MB = 15MB.
-- The storage cost ($0.000023/GB/month) is negligible.

CREATE TABLE analytics.dim_customer (
    customer_key      INTEGER IDENTITY(1,1) PRIMARY KEY,
    customer_id       VARCHAR(36)  NOT NULL UNIQUE ENCODE ZSTD,
    company_name      VARCHAR(200)          ENCODE ZSTD,
    plan_type         VARCHAR(50)           ENCODE BYTEDICT,
    industry          VARCHAR(100)          ENCODE ZSTD,
    country           CHAR(2)               ENCODE RAW,
    mrr_usd           DECIMAL(10,2)         ENCODE AZ64,
    signup_date       DATE                  ENCODE AZ64,
    account_manager   VARCHAR(100)          ENCODE ZSTD,
    is_active         BOOLEAN               DEFAULT TRUE
)
DISTSTYLE ALL;

-- ── DIM: Date ─────────────────────────────────────────────────
CREATE TABLE analytics.dim_date (
    date_key      INTEGER  PRIMARY KEY ENCODE AZ64,
    full_date     DATE     NOT NULL UNIQUE ENCODE AZ64,
    year_num      SMALLINT NOT NULL ENCODE AZ64,
    quarter_num   SMALLINT NOT NULL ENCODE AZ64,
    month_num     SMALLINT NOT NULL ENCODE AZ64,
    month_name    VARCHAR(10)       ENCODE BYTEDICT,
    week_num      SMALLINT NOT NULL ENCODE AZ64,
    day_of_week   SMALLINT NOT NULL ENCODE AZ64,
    day_name      VARCHAR(10)       ENCODE BYTEDICT,
    is_weekend    BOOLEAN  NOT NULL,
    is_holiday    BOOLEAN  NOT NULL DEFAULT FALSE
)
DISTSTYLE ALL;

-- ── Populate dim_date (2020-2030) ─────────────────────────────
INSERT INTO analytics.dim_date
SELECT
    TO_NUMBER(TO_CHAR(d, 'YYYYMMDD'), '99999999')::INT      AS date_key,
    d                                                        AS full_date,
    EXTRACT(YEAR  FROM d)::SMALLINT                         AS year_num,
    EXTRACT(QUARTER FROM d)::SMALLINT                       AS quarter_num,
    EXTRACT(MONTH FROM d)::SMALLINT                         AS month_num,
    TO_CHAR(d, 'Month')                                     AS month_name,
    EXTRACT(WEEK  FROM d)::SMALLINT                         AS week_num,
    EXTRACT(DOW   FROM d)::SMALLINT                         AS day_of_week,
    TO_CHAR(d, 'Day')                                       AS day_name,
    EXTRACT(DOW   FROM d) IN (0, 6)                         AS is_weekend,
    FALSE                                                    AS is_holiday
FROM (
    SELECT DATEADD(day, n, '2020-01-01'::DATE) AS d
    FROM (
        SELECT ROW_NUMBER() OVER (ORDER BY 1) - 1 AS n
        FROM stl_scan LIMIT 4018
    )
)
WHERE d <= '2030-12-31'::DATE;
```

### Step 3.3: Load Data with COPY

```python
import boto3

redshift_data = boto3.client("redshift-data", region_name="us-east-1")

def execute_redshift(sql: str, cluster_id: str = "cloudmetrics-prod", db: str = "analytics") -> str:
    """Execute SQL on Redshift and wait for completion."""
    response = redshift_data.execute_statement(
        ClusterIdentifier=cluster_id,
        Database=db,
        DbUser="admin",
        Sql=sql,
    )
    statement_id = response["Id"]

    import time
    while True:
        status = redshift_data.describe_statement(Id=statement_id)
        if status["Status"] in ("FINISHED", "FAILED", "ABORTED"):
            if status["Status"] != "FINISHED":
                raise RuntimeError(f"Query failed: {status.get('Error', 'Unknown error')}")
            return statement_id
        time.sleep(2)


# Idempotent daily load: delete then insert
process_date = "2026-03-19"

# Step 1: Delete today's existing rows (idempotency)
execute_redshift(f"""
    DELETE FROM analytics.fact_api_events
    WHERE event_date = '{process_date}'::DATE;
""")

# Step 2: COPY from S3 Parquet
# WHY MANIFEST: Prevents partial loads if new files arrive during COPY.
# The manifest explicitly lists exactly which files to load.
execute_redshift(f"""
    COPY analytics.fact_api_events (
        event_id, customer_id, event_type, endpoint, method,
        status_code, response_time_ms, request_size_bytes,
        response_size_bytes, event_timestamp, region, event_date
    )
    FROM 's3://cloudmetrics-data-lake-prod/manifests/events_{process_date}.manifest'
    IAM_ROLE 'arn:aws:iam::123456789012:role/cloudmetrics-redshift-spectrum-role'
    FORMAT AS PARQUET
    MANIFEST
    COMPUPDATE ON
    STATUPDATE ON;
""")

# Step 3: Check for load errors
errors_df = execute_redshift("""
    SELECT filename, colname, type, raw_field_value, err_reason
    FROM stl_load_errors
    WHERE starttime > GETDATE() - INTERVAL '1 hour'
    ORDER BY starttime DESC
    LIMIT 20;
""")
print(f"Load complete for {process_date}")
```

### Step 3.4: External Schema for Spectrum (Historical Queries)

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

-- Create external schema pointing to Glue Catalog
CREATE EXTERNAL SCHEMA IF NOT EXISTS spectrum_raw
FROM DATA CATALOG
DATABASE 'cloudmetrics_raw'
IAM_ROLE 'arn:aws:iam::123456789012:role/cloudmetrics-redshift-spectrum-role'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- Query 3 years of historical data from S3 alongside today's Redshift data
-- Spectrum handles the S3 read; Redshift handles the local data and the join
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
    -- Historical data (90 days to 7 years) — from S3 via Spectrum
    SELECT
        EXTRACT(YEAR  FROM event_timestamp)::INT AS year_num,
        EXTRACT(MONTH FROM event_timestamp)::INT AS month_num,
        COUNT(*)                                  AS event_count
    FROM spectrum_raw.events
    WHERE year BETWEEN '2020' AND '2024'   -- Partition pruning!
    GROUP BY 1, 2
) s ON r.year_num = s.year_num AND r.month_num = s.month_num
ORDER BY 1, 2;
```

---

## PART 4: IAM SECURITY MODEL (45 min)

### Step 4.1: Define All Service Roles

**WHY:** Each service in the pipeline needs only the permissions required to do its job. Overly permissive roles are the leading cause of data breaches in cloud environments. Four roles, four minimum permission sets.

```python
import boto3
import json

iam = boto3.client("iam")

def create_role(role_name: str, trust_service: str, description: str) -> str:
    """Create an IAM role with a service trust policy."""
    trust_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": f"{trust_service}.amazonaws.com"},
            "Action": "sts:AssumeRole",
        }]
    }
    response = iam.create_role(
        RoleName=role_name,
        AssumeRolePolicyDocument=json.dumps(trust_policy),
        Description=description,
    )
    return response["Role"]["Arn"]


# Role 1: Kinesis Producer (assumed by the CloudMetrics application)
producer_arn = create_role(
    "cloudmetrics-kinesis-producer-role",
    trust_service="ec2",
    description="CloudMetrics API server — write events to Kinesis only",
)
iam.put_role_policy(
    RoleName="cloudmetrics-kinesis-producer-role",
    PolicyName="KinesisProducerPolicy",
    PolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Sid": "KinesisWriteOnly",
            "Effect": "Allow",
            "Action": ["kinesis:PutRecord", "kinesis:PutRecords", "kinesis:DescribeStream"],
            "Resource": "arn:aws:kinesis:us-east-1:123456789012:stream/cloudmetrics-events",
        }]
    }),
)

# Role 2: Firehose Delivery Role
firehose_arn = create_role(
    "cloudmetrics-firehose-delivery-role",
    trust_service="firehose",
    description="Firehose: read from Kinesis, write to S3, convert via Glue",
)
iam.put_role_policy(
    RoleName="cloudmetrics-firehose-delivery-role",
    PolicyName="FirehoseDeliveryPolicy",
    PolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "KinesisReadSource",
                "Effect": "Allow",
                "Action": ["kinesis:GetRecords", "kinesis:GetShardIterator",
                           "kinesis:DescribeStream", "kinesis:ListShards"],
                "Resource": "arn:aws:kinesis:us-east-1:123456789012:stream/cloudmetrics-events",
            },
            {
                "Sid": "S3WriteRawZoneOnly",
                "Effect": "Allow",
                "Action": ["s3:PutObject", "s3:GetBucketLocation"],
                "Resource": [
                    "arn:aws:s3:::cloudmetrics-data-lake-prod",
                    "arn:aws:s3:::cloudmetrics-data-lake-prod/raw/*",
                    "arn:aws:s3:::cloudmetrics-data-lake-prod/rejected/firehose/*",
                ],
            },
            {
                "Sid": "GlueSchemaRead",
                "Effect": "Allow",
                "Action": ["glue:GetTable", "glue:GetTableVersion", "glue:GetTableVersions"],
                "Resource": [
                    "arn:aws:glue:us-east-1:123456789012:catalog",
                    "arn:aws:glue:us-east-1:123456789012:database/cloudmetrics_raw",
                    "arn:aws:glue:us-east-1:123456789012:table/cloudmetrics_raw/events",
                ],
            },
            {
                "Sid": "KMSEncrypt",
                "Effect": "Allow",
                "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
                "Resource": "arn:aws:kms:us-east-1:123456789012:key/*",
                "Condition": {"StringEquals": {"kms:ViaService": "s3.us-east-1.amazonaws.com"}},
            },
        ]
    }),
)

# Role 3: Glue ETL Role
create_role(
    "cloudmetrics-glue-etl-role",
    trust_service="glue",
    description="Glue ETL: read raw, write processed/curated, update catalog",
)
# (Attach policy with S3 read raw + write processed, Glue catalog read/write,
#  CloudWatch Logs write, KMS decrypt/encrypt — similar pattern as above)

# Role 4: Redshift Spectrum Role
create_role(
    "cloudmetrics-redshift-spectrum-role",
    trust_service="redshift",
    description="Redshift: read from S3 data lake, read Glue catalog",
)
# (Attach policy with S3 GetObject + ListBucket read-only on data lake,
#  Glue GetDatabase + GetTable + GetPartitions read-only — no write permissions)

print("All service roles created")
```

---

## PART 5: COST CONTROLS & MONITORING (30 min)

### Step 5.1: Budget Alert and CloudWatch Alarms

```python
import boto3

budgets = boto3.client("budgets", region_name="us-east-1")
cloudwatch = boto3.client("cloudwatch", region_name="us-east-1")

# Set a monthly budget with 80% and 100% alerts
budgets.create_budget(
    AccountId="123456789012",
    Budget={
        "BudgetName": "cloudmetrics-data-platform",
        "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
        "TimeUnit": "MONTHLY",
        "BudgetType": "COST",
        "CostFilters": {"TagKeyValue": ["user:CostCenter$data-platform"]},
    },
    NotificationsWithSubscribers=[
        {
            "Notification": {
                "NotificationType": "ACTUAL",
                "ComparisonOperator": "GREATER_THAN",
                "Threshold": 80.0,
                "ThresholdType": "PERCENTAGE",
            },
            "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "data-team@cloudmetrics.io"}],
        },
        {
            "Notification": {
                "NotificationType": "FORECASTED",
                "ComparisonOperator": "GREATER_THAN",
                "Threshold": 100.0,
                "ThresholdType": "PERCENTAGE",
            },
            "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "cto@cloudmetrics.io"}],
        },
    ],
)

# Kinesis: alert if records are > 15 minutes old (data freshness problem)
cloudwatch.put_metric_alarm(
    AlarmName="Kinesis-GetRecords-IteratorAge-High",
    MetricName="GetRecords.IteratorAgeMilliseconds",
    Namespace="AWS/Kinesis",
    Dimensions=[{"Name": "StreamName", "Value": "cloudmetrics-events"}],
    Statistic="Maximum",
    Period=300,
    EvaluationPeriods=2,
    Threshold=900_000,   # 15 minutes in milliseconds
    ComparisonOperator="GreaterThanThreshold",
    AlarmDescription="Kinesis consumers are falling behind — data is > 15 min stale",
    AlarmActions=["arn:aws:sns:us-east-1:123456789012:data-platform-alerts"],
)

# Redshift: alert if CPU > 80% for 10 minutes (cluster may need scaling)
cloudwatch.put_metric_alarm(
    AlarmName="Redshift-CPU-High",
    MetricName="CPUUtilization",
    Namespace="AWS/Redshift",
    Dimensions=[{"Name": "ClusterIdentifier", "Value": "cloudmetrics-prod"}],
    Statistic="Average",
    Period=300,
    EvaluationPeriods=2,
    Threshold=80.0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=["arn:aws:sns:us-east-1:123456789012:data-platform-alerts"],
)

print("Budget and alarms configured")
```

---

## PART 6: COST ANALYSIS (30 min)

### Step 6.1: Monthly Cost Comparison

```python
import pandas as pd

costs = [
    # Current State
    ("RDS PostgreSQL r6g.2xlarge (2TB GP3)",   2800, "Cannot scale past 2TB"),
    ("RDS Backups (2TB, 30-day retention)",       400, "Point-in-time recovery"),
    ("CURRENT TOTAL",                            3200, "No path to 50TB"),
    ("", None, ""),
    # Proposed — S3
    ("S3 Standard (500GB new raw/month)",          12, "~500GB/month new data"),
    ("S3 Standard-IA (1TB processed)",             13, "90+ day data auto-tiered"),
    ("S3 Glacier IR (2TB historical > 1yr)",        8, "Long-tail archive"),
    ("S3 API Requests (10M PUT, 50M GET)",          25, "Firehose writes, Glue/Spectrum reads"),
    # Proposed — Kinesis
    ("Kinesis ON_DEMAND (200M events/day)",        450, "Auto-scales, ~$0.08/GB"),
    ("Kinesis Firehose (6TB/month delivered)",     210, "$0.035/GB delivered to S3"),
    # Proposed — Redshift
    ("Redshift ra3.xlplus x3 (1-year RI)",        662, "37% savings over On-Demand ($1,056)"),
    ("Redshift Spectrum (2TB scanned/month)",       10, "$5/TB, Parquet minimises scan"),
    # Proposed — Glue
    ("Glue ETL (10 DPU × 2hr/day)",               264, "10 DPU × $0.44 × 2hr × 30 days"),
    ("Glue Crawlers (5 crawlers, daily)",           44, "5 × $0.44/DPU × 0.67hr × 30 days"),
    # Other
    ("CloudWatch monitoring + alarms",              50, "Custom metrics and dashboards"),
    ("Data Transfer (VPC endpoints = free for S3)", 20, "S3 VPC endpoint avoids NAT cost"),
    ("PROPOSED TOTAL",                           1768, "Scales to 50TB+ with minimal cost growth"),
]

df = pd.DataFrame(costs, columns=["Component", "Monthly Cost (USD)", "Notes"])
print(df.to_markdown(index=False))

print("\n" + "=" * 60)
print(f"Savings vs current state: ${3200 - 1768:,}/month (${(3200 - 1768) * 12:,}/year)")
print(f"Scale ceiling: Current 2TB vs Proposed UNLIMITED")
print("=" * 60)
```

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The Kinesis stream uses `customer_id` as the partition key. A new enterprise customer signs up with 10x the API call volume of any existing customer. What happens to the Kinesis stream's shard distribution, and how would you detect and fix it?

2. The S3 lifecycle policy moves raw data to STANDARD\_IA after 30 days. An analyst wants to query raw JSON files from 45 days ago using Athena. What happens, and is there any cost implication? Would the behaviour be different for Parquet files in the processed zone?

3. The Redshift Spectrum role has `s3:GetObject` on the entire data lake bucket. The security team flags this as overly permissive. Propose a more tightly scoped permission that still allows Spectrum to query the curated zone without exposing the raw zone.

4. The cost analysis shows the proposed architecture at $1,768/month vs $3,200/month today. The CTO asks: "What happens to the cost if customer count grows from 5,000 to 50,000?" Walk through which line items scale linearly with data volume and which are roughly fixed.

5. You need to re-run the daily COPY load for March 19 because the Glue job wrote corrupted Parquet files. The manifest for March 19 still points to the old files. Walk through the exact sequence of steps to: produce corrected files, update the manifest, re-run the COPY, and verify the result is correct.
