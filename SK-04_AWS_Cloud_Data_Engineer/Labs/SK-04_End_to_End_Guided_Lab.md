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
| **Why does the cost math matter?** | The CTO needs a cost justification before approving the migration. Showing the architecture is cheaper per month than today AND has unlimited scale vs a 2TB cap is how you get sign-off. |

---

## WHAT YOU WILL BUILD

```
CloudMetrics AWS Data Platform (one bucket per trainee: cloudmetrics-data-lake-<name>):
│
├── 1. S3 DATA LAKE (SSE-KMS with a customer managed key)
│   ├── raw/          (events from Kinesis, converted JSON → Parquet by Firehose)
│   ├── processed/    (Parquet, partitioned by year/month/day — for future Glue jobs)
│   ├── curated/      (star schema for Redshift Spectrum — for future Glue jobs)
│   ├── rejected/     (quality failures + Firehose errors)
│   ├── manifests/    (COPY manifests — exactly which files each load used)
│   └── temp/         (ETL working area, expires in 7 days)
│
├── 2. IAM SECURITY MODEL
│   ├── cloudmetrics-kinesis-producer-role
│   ├── cloudmetrics-firehose-delivery-role
│   ├── cloudmetrics-glue-etl-role
│   └── cloudmetrics-redshift-spectrum-role
│
├── 3. KINESIS PIPELINE
│   ├── cloudmetrics-events (Data Stream, ON_DEMAND)
│   ├── cloudmetrics_raw.events (Glue Data Catalog table — the schema Firehose and Spectrum use)
│   └── cloudmetrics-events-to-s3 (Firehose → JSON → Parquet, deployed with CloudFormation)
│
├── 4. REDSHIFT DATA WAREHOUSE (cluster cloudmetrics-prod, database analytics)
│   ├── staging.api_events_raw    (COPY landing table)
│   ├── analytics.fact_api_events (dist: customer_id, sort: event_timestamp)
│   ├── analytics.dim_customer    (dist: ALL — 5K rows)
│   ├── analytics.dim_date        (dist: ALL)
│   └── spectrum_raw (external schema over S3 for 7-year history)
│
├── 5. COST CONTROLS
│   ├── S3 lifecycle policy (Standard → IA → Glacier IR → Deep Archive)
│   ├── SNS topic data-platform-alerts
│   ├── AWS Budget alert (tag-filtered)
│   └── CloudWatch alarms (Kinesis iterator age, Redshift CPU)
│
└── 6. TEARDOWN (mandatory — Part 7)
```

### Time Estimate: 8–10 hours

---

## PART 0: BEFORE YOU START — COST, SETUP, AND HABITS (20 min)

### ⚠️ Cost warning — read this first

This lab creates **real, hourly-billed** resources. Approximate us-east-1 prices (always confirm in the [AWS Pricing Calculator](https://calculator.aws/)):

| Resource | Bills while idle? | Approximate cost |
|---|---|---|
| Redshift `ra3.xlplus`, 1 node | **Yes — every hour it exists and is not paused** | ~$1.09/hour (~$26/day) |
| Kinesis Data Stream, ON_DEMAND | **Yes** — per stream-hour | ~$0.04/hour (~$1/day) + $0.08/GB written |
| Kinesis Data Firehose | No (per GB delivered) | $0.029/GB + $0.018/GB for Parquet conversion — pennies here |
| KMS customer managed key | Yes, prorated | $1/month (stops once the key is scheduled for deletion) |
| Secrets Manager (Redshift managed admin password) | Yes, prorated | $0.40/month |
| S3, Glue Data Catalog, SNS, CloudWatch alarms, Budgets | Effectively no | pennies / free tier |

**Habits that keep this lab under ~$10:**

1. **Pause Redshift whenever you step away** (storage still bills, compute stops):
   `aws redshift pause-cluster --cluster-identifier cloudmetrics-prod` and later `aws redshift resume-cluster --cluster-identifier cloudmetrics-prod`.
2. **Delete the Kinesis stream and Firehose at the end of every session** — recreating them takes 2 minutes.
3. **Finish with Part 7 (Teardown)** the same day you finish the lab. Then check Billing → Bills the next day.
4. Keep the learner budget from granular Lab_00 ($5/$20 alerts) active.

### Step 0.1: Project folder, tools, and shared settings

If you completed the SK-04 granular labs, you already have all of this — just open Git Bash and run the three "every session" commands below.

**One-time setup** (details in granular Lab_00): AWS CLI v2 configured for a non-root IAM user with region `us-east-1`, Git for Windows (Git Bash), Python 3.12, and a project folder **outside OneDrive**: `C:\DataEngineering\sk04`.

```bash
# Git Bash — one time
cd /c/DataEngineering/sk04
python -m venv .venv
source .venv/Scripts/activate
python -m pip install boto3 pandas tabulate     # tabulate is required by pandas' to_markdown() in Part 6
mkdir -p scripts sql infra
aws sts get-caller-identity                     # must show an IAM user ARN, never :root
```

Create `scripts/cm_config.py` — **the single place where your name lives**. Every script in this lab imports it, and bash reads it too:

```python
# scripts/cm_config.py — shared settings for the SK-04 granular labs and guided lab
NAME = "francis"                 # <- your name: lowercase letters, digits, hyphens only
ALERT_EMAIL = "you@example.com"  # <- an inbox you can open (budget + alarm emails)

REGION = "us-east-1"
BUCKET = f"cloudmetrics-data-lake-{NAME}"    # bucket names are global, so yours must be unique
KMS_ALIAS = "alias/cloudmetrics-data-key"
STREAM_NAME = "cloudmetrics-events"
FIREHOSE_NAME = "cloudmetrics-events-to-s3"
FIREHOSE_STACK = "cloudmetrics-firehose"
GLUE_DATABASE = "cloudmetrics_raw"
GLUE_TABLE = "events"
CLUSTER_ID = "cloudmetrics-prod"
DB_NAME = "analytics"
DB_USER = "awsuser"
SUBNET_GROUP = "cloudmetrics-subnet-group"
SECURITY_GROUP = "cloudmetrics-redshift-sg"
SNS_TOPIC = "data-platform-alerts"
BUDGET_NAME = "cloudmetrics-data-platform"
ROLE_PRODUCER = "cloudmetrics-kinesis-producer-role"
ROLE_FIREHOSE = "cloudmetrics-firehose-delivery-role"
ROLE_GLUE = "cloudmetrics-glue-etl-role"
ROLE_REDSHIFT = "cloudmetrics-redshift-spectrum-role"
TAGS = {"CostCenter": "data-platform", "Environment": "learning"}


def account_id() -> str:
    """12-digit account ID of whoever is logged in (never hard-code it)."""
    import boto3
    return boto3.client("sts", region_name=REGION).get_caller_identity()["Account"]


if __name__ == "__main__":
    # `eval "$(python scripts/cm_config.py)"` in Git Bash exports these for bash scripts
    for key in ("NAME", "REGION", "BUCKET", "KMS_ALIAS", "STREAM_NAME", "FIREHOSE_NAME",
                "FIREHOSE_STACK", "GLUE_DATABASE", "GLUE_TABLE", "CLUSTER_ID", "ROLE_FIREHOSE"):
        print(f'export {key}="{globals()[key]}"')
    print(f'export ACCOUNT_ID="{account_id()}"')
```

**Every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"     # exports BUCKET, REGION, ACCOUNT_ID, ...
echo "$BUCKET in $REGION for account $ACCOUNT_ID"
```

**How to run the code in this lab:** `bash` blocks run in Git Bash from `C:\DataEngineering\sk04`. Each `python` block starts with a comment naming its file (e.g. `# scripts/02_lifecycle.py`) — save it there and run `python scripts/02_lifecycle.py`. SQL files go in `sql\`, CloudFormation templates in `infra\`.

---

## PART 1: S3 DATA LAKE SETUP (60 min)

### Step 1.1: Create and Configure the Data Lake Bucket

**WHY:** A bucket with no security configuration is a security incident waiting to happen. Enabling versioning, blocking public access, and applying KMS encryption should happen at creation time — retrofitting these later is operationally risky.

```bash
#!/bin/bash
# scripts/01_create_lake.sh — run in Git Bash: bash scripts/01_create_lake.sh
# ═══════════════════════════════════════════════════════════════
# CloudMetrics S3 Data Lake — Initial Setup
# Run once per environment (dev / staging / prod)
# ═══════════════════════════════════════════════════════════════
set -euo pipefail
eval "$(python scripts/cm_config.py)"      # BUCKET, REGION, KMS_ALIAS, ...

# 1. Create the bucket
# (us-east-1 is the one region where you must NOT pass --create-bucket-configuration)
aws s3api create-bucket \
    --bucket "$BUCKET" \
    --region "$REGION"

# 1b. Tag it — the Part 5 budget filters spend by CostCenter
aws s3api put-bucket-tagging \
    --bucket "$BUCKET" \
    --tagging 'TagSet=[{Key=CostCenter,Value=data-platform},{Key=Environment,Value=learning}]'

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

# 4. Create (or reuse) a Customer Managed Key (CMK) for encryption
# WHY CMK over SSE-S3: CMK gives us key rotation control,
# CloudTrail audit of every decrypt operation, and cross-account
# access management. $1/month per key.
if KEY_ID=$(aws kms describe-key --key-id "$KMS_ALIAS" --region "$REGION" \
                --query 'KeyMetadata.KeyId' --output text 2>/dev/null); then
    echo "Reusing existing key $KEY_ID ($KMS_ALIAS)"
else
    KEY_ID=$(aws kms create-key \
        --description "CloudMetrics data lake encryption key" \
        --tags TagKey=CostCenter,TagValue=data-platform \
        --region "$REGION" \
        --query 'KeyMetadata.KeyId' --output text)
    aws kms create-alias --alias-name "$KMS_ALIAS" --target-key-id "$KEY_ID" --region "$REGION"
    echo "Created key $KEY_ID with alias $KMS_ALIAS"
fi
aws kms enable-key-rotation --key-id "$KEY_ID" --region "$REGION"   # yearly automatic rotation
KEY_ARN=$(aws kms describe-key --key-id "$KEY_ID" --region "$REGION" \
              --query 'KeyMetadata.Arn' --output text)

# 5. Enable default KMS encryption (Bucket Key = far fewer, cheaper KMS calls)
aws s3api put-bucket-encryption \
    --bucket "$BUCKET" \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "'"$KEY_ARN"'"
            },
            "BucketKeyEnabled": true
        }]
    }'

echo "S3 bucket $BUCKET configured successfully"
```

**Expected output** (IDs differ):

```
{
    "Location": "/cloudmetrics-data-lake-francis"
}
Created key 1234abcd-12ab-34cd-56ef-1234567890ab with alias alias/cloudmetrics-data-key
S3 bucket cloudmetrics-data-lake-francis configured successfully
```

Verify: `aws s3api get-bucket-encryption --bucket "$BUCKET"` shows `"SSEAlgorithm": "aws:kms"` and `"BucketKeyEnabled": true`.

### Step 1.2: Create Lifecycle Policy

**WHY:** Without a lifecycle policy, every byte of raw data stays in S3 Standard ($23/TB/month) forever. With the policy below, data automatically migrates to cheaper tiers as it ages — raw data that was accessed daily in week 1 costs $1/TB/month after 1 year.

```python
# scripts/02_lifecycle.py
import boto3

from cm_config import BUCKET, REGION

s3 = boto3.client("s3", region_name=REGION)

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
    Bucket=BUCKET,
    LifecycleConfiguration=lifecycle_policy,
)
print("Lifecycle policy applied")
```

**Expected output:** `Lifecycle policy applied`. Verify with `aws s3api get-bucket-lifecycle-configuration --bucket "$BUCKET"` (4 rules).

---

## PART 2: IAM SECURITY MODEL (45 min)

> Built *before* Kinesis, Firehose and Redshift because each of those services needs its role to exist when it is created.

### Step 2.1: Define All Service Roles

**WHY:** Each service in the pipeline needs only the permissions required to do its job. Overly permissive roles are the leading cause of data breaches in cloud environments. Four roles, four minimum permission sets.

```python
# scripts/03_iam_roles.py
import json
import time

import boto3

from cm_config import (BUCKET, GLUE_DATABASE, GLUE_TABLE, KMS_ALIAS, REGION, ROLE_FIREHOSE,
                       ROLE_GLUE, ROLE_PRODUCER, ROLE_REDSHIFT, STREAM_NAME, TAGS, account_id)

iam = boto3.client("iam")
ACCOUNT = account_id()
KEY_ARN = boto3.client("kms", region_name=REGION).describe_key(KeyId=KMS_ALIAS)["KeyMetadata"]["Arn"]
STREAM_ARN = f"arn:aws:kinesis:{REGION}:{ACCOUNT}:stream/{STREAM_NAME}"
BUCKET_ARN = f"arn:aws:s3:::{BUCKET}"
GLUE_CATALOG = f"arn:aws:glue:{REGION}:{ACCOUNT}:catalog"
GLUE_DB = f"arn:aws:glue:{REGION}:{ACCOUNT}:database/{GLUE_DATABASE}"
GLUE_TABLES = f"arn:aws:glue:{REGION}:{ACCOUNT}:table/{GLUE_DATABASE}/*"


def create_role(role_name: str, trust_service: str, description: str) -> str:
    """Create an IAM role with a service trust policy (re-running is safe)."""
    trust_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": f"{trust_service}.amazonaws.com"},
            "Action": "sts:AssumeRole",
        }]
    }
    try:
        response = iam.create_role(
            RoleName=role_name,
            AssumeRolePolicyDocument=json.dumps(trust_policy),
            Description=description,
            Tags=[{"Key": k, "Value": v} for k, v in TAGS.items()],
        )
        print(f"Created {role_name}")
    except iam.exceptions.EntityAlreadyExistsException:
        response = iam.get_role(RoleName=role_name)
        print(f"Reusing {role_name}")
    return response["Role"]["Arn"]


def put_policy(role_name: str, policy_name: str, statements: list) -> None:
    """Attach (or overwrite) the role's inline permission policy."""
    iam.put_role_policy(
        RoleName=role_name,
        PolicyName=policy_name,
        PolicyDocument=json.dumps({"Version": "2012-10-17", "Statement": statements}),
    )


# Role 1: Kinesis Producer (assumed by the CloudMetrics application servers)
create_role(ROLE_PRODUCER, trust_service="ec2",
            description="CloudMetrics API server — write events to Kinesis only")
put_policy(ROLE_PRODUCER, "KinesisProducerPolicy", [{
    "Sid": "KinesisWriteOnly",
    "Effect": "Allow",
    "Action": ["kinesis:PutRecord", "kinesis:PutRecords", "kinesis:DescribeStream"],
    "Resource": STREAM_ARN,
}])

# Role 2: Firehose Delivery Role — used for the Kinesis source, the S3 destination
# AND the Glue schema lookup in the Part 3 CloudFormation template
create_role(ROLE_FIREHOSE, trust_service="firehose",
            description="Firehose: read from Kinesis, write to S3, convert via Glue")
put_policy(ROLE_FIREHOSE, "FirehoseDeliveryPolicy", [
    {
        "Sid": "KinesisReadSource",
        "Effect": "Allow",
        "Action": ["kinesis:GetRecords", "kinesis:GetShardIterator", "kinesis:DescribeStream",
                   "kinesis:DescribeStreamSummary", "kinesis:ListShards"],
        "Resource": STREAM_ARN,
    },
    {
        "Sid": "S3WriteRawZoneOnly",
        "Effect": "Allow",
        "Action": ["s3:AbortMultipartUpload", "s3:GetBucketLocation", "s3:GetObject",
                   "s3:ListBucket", "s3:ListBucketMultipartUploads", "s3:PutObject"],
        "Resource": [
            BUCKET_ARN,
            f"{BUCKET_ARN}/raw/*",
            f"{BUCKET_ARN}/rejected/firehose/*",
        ],
    },
    {
        "Sid": "GlueSchemaRead",
        "Effect": "Allow",
        "Action": ["glue:GetTable", "glue:GetTableVersion", "glue:GetTableVersions"],
        "Resource": [GLUE_CATALOG, GLUE_DB, f"arn:aws:glue:{REGION}:{ACCOUNT}:table/{GLUE_DATABASE}/{GLUE_TABLE}"],
    },
    {
        "Sid": "KMSEncrypt",
        "Effect": "Allow",
        "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
        "Resource": KEY_ARN,
        "Condition": {"StringEquals": {"kms:ViaService": f"s3.{REGION}.amazonaws.com"}},
    },
])

# Role 3: Glue ETL Role (for the Glue jobs that will build processed/ and curated/)
create_role(ROLE_GLUE, trust_service="glue",
            description="Glue ETL: read raw, write processed/curated, update catalog")
put_policy(ROLE_GLUE, "GlueEtlPolicy", [
    {"Sid": "ListLake", "Effect": "Allow",
     "Action": ["s3:ListBucket", "s3:GetBucketLocation"], "Resource": BUCKET_ARN},
    {"Sid": "ReadRaw", "Effect": "Allow",
     "Action": ["s3:GetObject"], "Resource": f"{BUCKET_ARN}/raw/*"},
    {"Sid": "WriteDownstreamZones", "Effect": "Allow",
     "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
     "Resource": [f"{BUCKET_ARN}/{zone}/*" for zone in ("processed", "curated", "rejected", "temp")]},
    {"Sid": "CatalogReadWrite", "Effect": "Allow",
     "Action": ["glue:GetDatabase", "glue:GetDatabases", "glue:GetTable", "glue:GetTables",
                "glue:CreateTable", "glue:UpdateTable", "glue:GetPartition", "glue:GetPartitions",
                "glue:CreatePartition", "glue:BatchCreatePartition", "glue:UpdatePartition"],
     "Resource": [GLUE_CATALOG, f"arn:aws:glue:{REGION}:{ACCOUNT}:database/cloudmetrics_*",
                  f"arn:aws:glue:{REGION}:{ACCOUNT}:table/cloudmetrics_*/*"]},
    {"Sid": "GlueLogs", "Effect": "Allow",
     "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
     "Resource": f"arn:aws:logs:{REGION}:{ACCOUNT}:log-group:/aws-glue/*"},
    {"Sid": "KMSForLake", "Effect": "Allow",
     "Action": ["kms:Decrypt", "kms:Encrypt", "kms:GenerateDataKey"], "Resource": KEY_ARN},
])

# Role 4: Redshift Spectrum Role (COPY + Spectrum) — READ-ONLY on the lake
create_role(ROLE_REDSHIFT, trust_service="redshift",
            description="Redshift: read from S3 data lake, read Glue catalog")
put_policy(ROLE_REDSHIFT, "RedshiftLakeReadOnly", [
    {"Sid": "ListLake", "Effect": "Allow",
     "Action": ["s3:ListBucket", "s3:GetBucketLocation"], "Resource": BUCKET_ARN},
    {"Sid": "ReadLakeObjects", "Effect": "Allow",
     "Action": ["s3:GetObject"], "Resource": f"{BUCKET_ARN}/*"},
    {"Sid": "CatalogReadOnly", "Effect": "Allow",
     "Action": ["glue:GetDatabase", "glue:GetDatabases", "glue:GetTable", "glue:GetTables",
                "glue:GetPartition", "glue:GetPartitions", "glue:BatchGetPartition"],
     "Resource": [GLUE_CATALOG, GLUE_DB, GLUE_TABLES,
                  f"arn:aws:glue:{REGION}:{ACCOUNT}:database/default"]},
    {"Sid": "KMSDecrypt", "Effect": "Allow",
     "Action": ["kms:Decrypt"], "Resource": KEY_ARN},
])

time.sleep(10)   # IAM is eventually consistent — give new roles a moment before services use them
print("All service roles created")
```

**Expected output:**

```
Created cloudmetrics-kinesis-producer-role
Created cloudmetrics-firehose-delivery-role
Created cloudmetrics-glue-etl-role
Created cloudmetrics-redshift-spectrum-role
All service roles created
```

**Note:** the KMS key's default key policy delegates access to IAM, so the `kms:*` statements in these role policies are what grant (or deny) each service use of the key.

---

## PART 3: KINESIS INGESTION PIPELINE (60 min)

### Step 3.1: Create the Data Stream

**WHY:** ON\_DEMAND mode auto-scales shards based on actual traffic. CloudMetrics has variable traffic (10x spike during US business hours). The $0.08/GB cost is predictable and eliminates the 3 AM "add shards before the morning peak" operational burden.

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

**Expected output:** `Stream cloudmetrics-events is ACTIVE` (after ~30 seconds). ⚠️ The stream now bills per hour — finish Part 3 in one sitting or delete it (Part 7).

### Step 3.2: Register the Event Schema in the Glue Data Catalog

**WHY:** Firehose cannot convert JSON to Parquet without a schema, and Redshift Spectrum cannot query S3 without a table definition. Both read the same Glue Data Catalog table — one schema, two consumers. The table is partitioned exactly like the Firehose prefix in Step 3.3.

```python
# scripts/05_glue_catalog.py
import boto3

from cm_config import BUCKET, GLUE_DATABASE, GLUE_TABLE, REGION

glue = boto3.client("glue", region_name=REGION)

try:
    glue.create_database(DatabaseInput={
        "Name": GLUE_DATABASE,
        "Description": "CloudMetrics raw zone (Firehose output)",
    })
except glue.exceptions.AlreadyExistsException:
    pass

columns = [
    {"Name": "event_id",            "Type": "string"},
    {"Name": "customer_id",         "Type": "string"},
    {"Name": "event_type",          "Type": "string"},
    {"Name": "endpoint",            "Type": "string"},
    {"Name": "method",              "Type": "string"},
    {"Name": "status_code",         "Type": "int"},
    {"Name": "response_time_ms",    "Type": "int"},
    {"Name": "request_size_bytes",  "Type": "bigint"},
    {"Name": "response_size_bytes", "Type": "bigint"},
    {"Name": "event_timestamp",     "Type": "timestamp"},
    {"Name": "region",              "Type": "string"},
]

table_input = {
    "Name": GLUE_TABLE,
    "TableType": "EXTERNAL_TABLE",
    "Parameters": {"classification": "parquet", "EXTERNAL": "TRUE"},
    "PartitionKeys": [{"Name": p, "Type": "string"} for p in ("year", "month", "day", "hour")],
    "StorageDescriptor": {
        "Columns": columns,
        "Location": f"s3://{BUCKET}/raw/events/",
        "InputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
        "OutputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
        "SerdeInfo": {
            "SerializationLibrary": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe",
            "Parameters": {"serialization.format": "1"},
        },
    },
}

try:
    glue.create_table(DatabaseName=GLUE_DATABASE, TableInput=table_input)
    print(f"Created table {GLUE_DATABASE}.{GLUE_TABLE}")
except glue.exceptions.AlreadyExistsException:
    glue.update_table(DatabaseName=GLUE_DATABASE, TableInput=table_input)
    print(f"Updated table {GLUE_DATABASE}.{GLUE_TABLE}")
```

**Expected output:** `Created table cloudmetrics_raw.events`.

### Step 3.3: Firehose Delivery Stream (CloudFormation)

**WHY:** Firehose is defined as CloudFormation rather than boto3 because it has many interdependent configuration blocks. CloudFormation validates the full configuration as a unit and enables version-controlled re-deployment. The template takes your bucket and the Part 2 role as **parameters**, so the same file deploys for every trainee and every environment.

Save as `infra/firehose.json`:

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "CloudMetrics Kinesis Firehose → S3 with Parquet conversion",

    "Parameters": {
        "BucketName":      {"Type": "String", "Description": "cloudmetrics-data-lake-<name>"},
        "FirehoseRoleArn": {"Type": "String", "Description": "ARN of cloudmetrics-firehose-delivery-role (Part 2)"},
        "StreamName":      {"Type": "String", "Default": "cloudmetrics-events"},
        "GlueDatabase":    {"Type": "String", "Default": "cloudmetrics_raw"},
        "GlueTable":       {"Type": "String", "Default": "events"}
    },

    "Resources": {
        "EventsFirehose": {
            "Type": "AWS::KinesisFirehose::DeliveryStream",
            "Properties": {
                "DeliveryStreamName": "cloudmetrics-events-to-s3",
                "DeliveryStreamType": "KinesisStreamAsSource",
                "KinesisStreamSourceConfiguration": {
                    "KinesisStreamARN": {
                        "Fn::Sub": "arn:aws:kinesis:${AWS::Region}:${AWS::AccountId}:stream/${StreamName}"
                    },
                    "RoleARN": {"Ref": "FirehoseRoleArn"}
                },
                "ExtendedS3DestinationConfiguration": {
                    "BucketARN": {
                        "Fn::Sub": "arn:aws:s3:::${BucketName}"
                    },
                    "Prefix": "raw/events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
                    "ErrorOutputPrefix": "rejected/firehose/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
                    "BufferingHints": {
                        "SizeInMBs": 128,
                        "IntervalInSeconds": 300
                    },
                    "RoleARN": {"Ref": "FirehoseRoleArn"},
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
                            "DatabaseName": {"Ref": "GlueDatabase"},
                            "TableName":    {"Ref": "GlueTable"},
                            "Region":       {"Ref": "AWS::Region"},
                            "RoleARN":      {"Ref": "FirehoseRoleArn"},
                            "VersionId":    "LATEST"
                        }
                    }
                },
                "Tags": [
                    {"Key": "CostCenter",  "Value": "data-platform"},
                    {"Key": "Environment", "Value": "learning"}
                ]
            }
        }
    },

    "Outputs": {
        "DeliveryStreamArn": {"Value": {"Fn::GetAtt": ["EventsFirehose", "Arn"]}}
    }
}
```

Deploy it (Git Bash, after the "every session" commands):

```bash
aws cloudformation validate-template --template-body file://infra/firehose.json
aws cloudformation deploy \
    --template-file infra/firehose.json \
    --stack-name cloudmetrics-firehose \
    --region "$REGION" \
    --parameter-overrides \
        BucketName="$BUCKET" \
        FirehoseRoleArn="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_FIREHOSE}"
aws firehose describe-delivery-stream --delivery-stream-name cloudmetrics-events-to-s3 \
    --query 'DeliveryStreamDescription.DeliveryStreamStatus'
```

**Expected output:** `Successfully created/updated stack - cloudmetrics-firehose`, then `"ACTIVE"`.

**Note:** the S3 objects Firehose writes are encrypted automatically by the bucket's default SSE-KMS setting (Part 1) — that is why the Firehose role needs `kms:GenerateDataKey`.

### Step 3.4: Resilient Event Producer

**WHY:** PutRecords is the right API for batch sends — one call for up to 500 records instead of 500 calls. But PutRecords has partial failures: some records succeed, some fail. The code below handles partial failures with exponential backoff rather than silently dropping records.

> Run this **after** Step 3.3: Firehose starts reading the stream at the moment it is created, so events sent earlier never reach S3.

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

**Expected output:** `Sent: 10,000 | Failed: 0`.

Firehose flushes when 128 MB **or** 300 seconds is reached — with 3.5 MB of test data, wait ~5 minutes, then:

```bash
aws s3 ls "s3://$BUCKET/raw/events/" --recursive
aws s3 ls "s3://$BUCKET/rejected/" --recursive     # should be empty
```

**Expected output:** one or more objects like `raw/events/year=2026/month=03/day=19/hour=14/cloudmetrics-events-to-s3-1-2026-03-19-14-05-12-<random-id>` (Parquet content; Firehose adds no file extension by default). Objects under `rejected/firehose/format-conversion-failed/` mean the JSON did not match the Glue table — compare field names and types.

---

## PART 4: REDSHIFT DATA WAREHOUSE (90 min)

### Step 4.1: Create the Network Pieces and the Cluster

**WHY the network pieces:** a provisioned cluster lives inside a VPC. A **cluster subnet group** tells Redshift which subnets it may use; a **security group** with *no inbound rules* plus `PubliclyAccessible=False` means nothing on the internet can connect. We still reach the cluster through the **Redshift Data API** (an HTTPS AWS API authorised by IAM), so no ports need opening. **WHY a managed password:** `ManageMasterPassword=True` makes Redshift generate the admin password and keep it in **Secrets Manager** — no password ever appears in code, chat, or Git.

```python
# scripts/07_create_cluster.py
import boto3

from cm_config import (CLUSTER_ID, DB_NAME, DB_USER, KMS_ALIAS, REGION, ROLE_REDSHIFT,
                       SECURITY_GROUP, SUBNET_GROUP, TAGS, account_id)

ec2 = boto3.client("ec2", region_name=REGION)
redshift = boto3.client("redshift", region_name=REGION)
kms = boto3.client("kms", region_name=REGION)

# 1. Network: use the account's default VPC and its default subnets
vpc_id = ec2.describe_vpcs(Filters=[{"Name": "isDefault", "Values": ["true"]}])["Vpcs"][0]["VpcId"]
subnet_ids = [s["SubnetId"] for s in ec2.describe_subnets(
    Filters=[{"Name": "vpc-id", "Values": [vpc_id]},
             {"Name": "default-for-az", "Values": ["true"]}])["Subnets"]]

try:
    redshift.create_cluster_subnet_group(
        ClusterSubnetGroupName=SUBNET_GROUP,
        Description="CloudMetrics Redshift subnets",
        SubnetIds=subnet_ids,
    )
except redshift.exceptions.ClusterSubnetGroupAlreadyExistsFault:
    pass

existing = ec2.describe_security_groups(Filters=[{"Name": "group-name", "Values": [SECURITY_GROUP]},
                                                 {"Name": "vpc-id", "Values": [vpc_id]}])["SecurityGroups"]
sg_id = existing[0]["GroupId"] if existing else ec2.create_security_group(
    GroupName=SECURITY_GROUP,
    Description="CloudMetrics Redshift - no inbound rules (Data API only)",
    VpcId=vpc_id,
)["GroupId"]

# 2. The cluster
# WHY ra3.xlplus:
#   - Managed storage: data in S3 RMS (unlimited), hot cache in local NVMe
#   - Production sizing for CloudMetrics: 3 nodes = 6 slices (2 slices per ra3.xlplus)
#   - THIS LAB builds 1 node (2 slices): identical SQL, one third of the cost
#   - Price: check the AWS Pricing Calculator (~$1.09/node-hour On-Demand in us-east-1 at time of writing);
#     Reserved Instances cut that substantially for always-on production clusters
role_arn = f"arn:aws:iam::{account_id()}:role/{ROLE_REDSHIFT}"
key_arn = kms.describe_key(KeyId=KMS_ALIAS)["KeyMetadata"]["Arn"]

try:
    response = redshift.create_cluster(
        ClusterIdentifier=CLUSTER_ID,
        NodeType="ra3.xlplus",
        ClusterType="single-node",          # production: ClusterType="multi-node", NumberOfNodes=3
        DBName=DB_NAME,
        MasterUsername=DB_USER,
        ManageMasterPassword=True,          # password generated + stored in Secrets Manager
        ClusterSubnetGroupName=SUBNET_GROUP,
        VpcSecurityGroupIds=[sg_id],
        PubliclyAccessible=False,
        IamRoles=[role_arn],
        DefaultIamRoleArn=role_arn,         # lets SQL say IAM_ROLE default
        Encrypted=True,
        KmsKeyId=key_arn,
        Tags=[{"Key": k, "Value": v} for k, v in TAGS.items()],
    )
    print(f"Cluster creation initiated: {response['Cluster']['ClusterStatus']}")
except redshift.exceptions.ClusterAlreadyExistsFault:
    print(f"Cluster {CLUSTER_ID} already exists")

print("Waiting for the cluster to become available (10-15 minutes)...")
redshift.get_waiter("cluster_available").wait(ClusterIdentifier=CLUSTER_ID,
                                             WaiterConfig={"Delay": 30, "MaxAttempts": 60})
print(f"Cluster {CLUSTER_ID} is available — it now bills by the hour. Pause it when you step away.")
```

**Expected output:**

```
Cluster creation initiated: creating
Waiting for the cluster to become available (10-15 minutes)...
Cluster cloudmetrics-prod is available — it now bills by the hour. Pause it when you step away.
```

### Step 4.2: Table Design

First save the small Data API helper used by every remaining step. It runs SQL through IAM — no drivers, passwords, or open ports:

```python
# scripts/rs.py — run SQL on the cluster through the Redshift Data API
import sys
import time

import boto3

from cm_config import CLUSTER_ID, DB_NAME, DB_USER, REGION

client = boto3.client("redshift-data", region_name=REGION)
TARGET = {"ClusterIdentifier": CLUSTER_ID, "Database": DB_NAME, "DbUser": DB_USER}


def _wait(statement_id: str) -> dict:
    """Poll until the statement finishes; raise with Redshift's error message if it failed."""
    while True:
        desc = client.describe_statement(Id=statement_id)
        if desc["Status"] in ("FINISHED", "FAILED", "ABORTED"):
            if desc["Status"] != "FINISHED":
                raise RuntimeError(f"Query failed: {desc.get('Error', 'Unknown error')}")
            return desc
        time.sleep(2)


def _rows(statement_id: str) -> tuple:
    """Fetch a finished statement's result set as (column_names, rows)."""
    result = client.get_statement_result(Id=statement_id)
    columns = [c["name"] for c in result["ColumnMetadata"]]
    rows = [[None if cell.get("isNull") else next(iter(cell.values())) for cell in record]
            for record in result["Records"]]
    return columns, rows


def run(sql: str):
    """Run ONE statement (auto-commit). Returns (columns, rows) for queries, else None."""
    statement_id = client.execute_statement(Sql=sql, **TARGET)["Id"]
    desc = _wait(statement_id)
    return _rows(statement_id) if desc.get("HasResultSet") else None


def run_transaction(statements: list) -> list:
    """Run several statements as ONE transaction: all succeed, or all roll back."""
    statement_id = client.batch_execute_statement(Sqls=statements, **TARGET)["Id"]
    desc = _wait(statement_id)
    return [_rows(sub["Id"]) if sub.get("HasResultSet") else None
            for sub in desc.get("SubStatements", [])]


def show(result) -> None:
    if result:
        columns, rows = result
        print(" | ".join(columns))
        for row in rows:
            print(" | ".join(str(v) for v in row))


def split_sql(text: str) -> list:
    """Split a .sql file into statements (drops full-line -- comments)."""
    lines = [line for line in text.splitlines() if not line.strip().startswith("--")]
    return [s.strip() for s in "\n".join(lines).split(";") if s.strip()]


if __name__ == "__main__":
    with open(sys.argv[1], encoding="utf-8") as f:
        for statement in split_sql(f.read()):
            print(f"-> {statement.splitlines()[0][:70]}")
            show(run(statement))
```

**WHY each design choice is documented in comments — distribution and sort key decisions are the most consequential performance choices in Redshift and should be version-controlled alongside the DDL.**

Save as `sql/01_schema.sql` and run `python scripts/rs.py sql/01_schema.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- CloudMetrics Redshift Schema  (safe to re-run)
-- ═══════════════════════════════════════════════════════════════

CREATE SCHEMA IF NOT EXISTS analytics;
CREATE SCHEMA IF NOT EXISTS staging;

-- ── FACT: API Events ─────────────────────────────────────────
-- Volume: 200M events/day → 6B rows/month
-- DISTKEY(customer_id): Most queries filter/group by customer.
--   5,000 unique customers = even distribution across 6 slices
--   (~1,000 customers per slice) in production. JOIN with dim_customer is local.
-- SORTKEY(event_timestamp): >80% of queries have a date range filter.
--   Zone maps allow Redshift to skip entire 1MB blocks outside the range.
-- AZ64 encoding for all numeric/timestamp columns (AWS default, good general purpose).
-- BYTEDICT for method/region (< 20 unique values = dictionary perfect fit).
-- ZSTD for customer_id/endpoint (variable length strings, high compression).

CREATE TABLE IF NOT EXISTS analytics.fact_api_events (
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

-- ── STAGING: exact shape of the Firehose Parquet files ───────
-- COPY from Parquet maps columns by position, so this table has the same 11 columns,
-- in the same order, as the Glue table cloudmetrics_raw.events.
CREATE TABLE IF NOT EXISTS staging.api_events_raw (
    event_id              VARCHAR(36),
    customer_id           VARCHAR(36),
    event_type            VARCHAR(50),
    endpoint              VARCHAR(500),
    method                VARCHAR(10),
    status_code           INTEGER,
    response_time_ms      INTEGER,
    request_size_bytes    BIGINT,
    response_size_bytes   BIGINT,
    event_timestamp       TIMESTAMP,
    region                VARCHAR(20)
);

-- ── DIM: Customer ─────────────────────────────────────────────
-- Volume: 5,000 rows — tiny
-- DISTSTYLE ALL: replicate the full 5K-row table to every node.
-- WHY: dim_customer is joined in almost every query.
--   With ALL distribution, the JOIN is always local — no network shuffle.
-- At 5K rows × ~500 bytes = 2.5MB total. ALL copies = 3 nodes × 2.5MB = 7.5MB.
-- The storage cost is negligible.

CREATE TABLE IF NOT EXISTS analytics.dim_customer (
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
CREATE TABLE IF NOT EXISTS analytics.dim_date (
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

-- ── Populate dim_date (2020-2030) — idempotent: clear, then rebuild ─────
-- The number generator cross-joins four 10-row lists = 10,000 numbers (0-9999),
-- which works on a brand-new cluster (system tables can be almost empty there).
DELETE FROM analytics.dim_date;

INSERT INTO analytics.dim_date
SELECT
    TO_NUMBER(TO_CHAR(d, 'YYYYMMDD'), '99999999')::INT      AS date_key,
    d                                                        AS full_date,
    EXTRACT(YEAR  FROM d)::SMALLINT                         AS year_num,
    EXTRACT(QUARTER FROM d)::SMALLINT                       AS quarter_num,
    EXTRACT(MONTH FROM d)::SMALLINT                         AS month_num,
    TRIM(TO_CHAR(d, 'Month'))                               AS month_name,
    EXTRACT(WEEK  FROM d)::SMALLINT                         AS week_num,
    EXTRACT(DOW   FROM d)::SMALLINT                         AS day_of_week,
    TRIM(TO_CHAR(d, 'Day'))                                 AS day_name,
    EXTRACT(DOW   FROM d) IN (0, 6)                         AS is_weekend,
    FALSE                                                    AS is_holiday
FROM (
    SELECT DATEADD(day, n, '2020-01-01'::DATE)::DATE AS d
    FROM (
        SELECT a.i + b.i * 10 + c.i * 100 + e.i * 1000 AS n
        FROM (SELECT 0 AS i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
              UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) a
        CROSS JOIN (SELECT 0 AS i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
              UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) b
        CROSS JOIN (SELECT 0 AS i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
              UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) c
        CROSS JOIN (SELECT 0 AS i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
              UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) e
    ) numbers
) dates
WHERE d <= '2030-12-31'::DATE;

SELECT COUNT(*) AS dim_date_rows FROM analytics.dim_date;
```

**Expected output:** one `->` line per statement, ending with:

```
dim_date_rows
4018
```

### Step 4.3: Load Data with COPY

**WHY a manifest:** it prevents partial loads if new files arrive during COPY — the manifest lists exactly which files to load, and it stays in S3 as an audit record of what each load used. **WHY staging + one transaction:** COPY lands the Parquet files as-is in `staging`; the same transaction then deletes the day from the fact table and inserts it again, so a re-run can never duplicate rows and a failure leaves the fact table untouched.

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

**Expected output** (for the 10,000 test events):

```
Manifest s3://cloudmetrics-data-lake-francis/manifests/events_2026-03-19.manifest lists 1 file(s)
rows_loaded
10000
query_id | table_name | status | loaded_rows | start_time
...      | api_events_raw | completed | 10000 | 2026-03-19 14:20:03.123
Load complete for 2026-03-19
```

`stl_load_errors` prints nothing when the load is clean. (A Parquet COPY with a column mismatch fails the whole statement with the error message raised by `rs.py`; `stl_load_errors` rows mostly appear for CSV/JSON loads.) Run the script a second time: `rows_loaded` stays **10000** — that is idempotency.

### Step 4.4: External Schema for Spectrum (Historical Queries)

Spectrum only sees partitions that are registered in the Glue Data Catalog. Firehose writes new `year=/month=/day=/hour=` folders but does not register them, so register them first:

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

**Expected output:** `1 partition(s) registered in cloudmetrics_raw.events` (one per hour folder).

Save as `sql/02_spectrum.sql` and run `python scripts/rs.py sql/02_spectrum.sql`:

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

**Expected output** (one row per month you have data for):

```
year | month | redshift_count | spectrum_count
2026 | 3     | 10000          | 10000
```

> **Leaving for a while?** `aws redshift pause-cluster --cluster-identifier cloudmetrics-prod` now, `resume-cluster` when you return (~5–10 minutes).

---

## PART 5: COST CONTROLS & MONITORING (30 min)

### Step 5.1: Alert Topic, Budget Alert and CloudWatch Alarms

```python
# scripts/10_cost_controls.py
import boto3

from cm_config import (ALERT_EMAIL, BUDGET_NAME, CLUSTER_ID, REGION, SNS_TOPIC, STREAM_NAME,
                       account_id)

ACCOUNT = account_id()
sns = boto3.client("sns", region_name=REGION)
budgets = boto3.client("budgets", region_name="us-east-1")   # Budgets is a global service
ce = boto3.client("ce", region_name="us-east-1")
cloudwatch = boto3.client("cloudwatch", region_name=REGION)

# 1. Alert channel: an SNS topic with your email subscribed (create_topic is idempotent)
topic_arn = sns.create_topic(Name=SNS_TOPIC, Tags=[{"Key": "CostCenter", "Value": "data-platform"}])["TopicArn"]
sns.subscribe(TopicArn=topic_arn, Protocol="email", Endpoint=ALERT_EMAIL)
print(f"Topic {topic_arn} — confirm the subscription email in {ALERT_EMAIL}")

# 2. Activate the CostCenter tag for cost allocation, so budgets/Cost Explorer can filter on it
#    (the tag only becomes activatable after billing has seen it on a resource — up to 24h)
try:
    ce.update_cost_allocation_tags_status(
        CostAllocationTagsStatus=[{"TagKey": "CostCenter", "Status": "Active"}])
    print("Cost allocation tag CostCenter: Active")
except Exception as exc:   # noqa: BLE001 — tag not seen by billing yet
    print(f"Could not activate CostCenter yet ({exc}); re-run this script tomorrow")

# 3. Monthly budget for everything tagged CostCenter=data-platform, with 80% and 100% alerts
BUDGET_USD = "20"   # CloudMetrics production uses 5000 — keep it small in a learning account
try:
    budgets.create_budget(
        AccountId=ACCOUNT,
        Budget={
            "BudgetName": BUDGET_NAME,
            "BudgetLimit": {"Amount": BUDGET_USD, "Unit": "USD"},
            "TimeUnit": "MONTHLY",
            "BudgetType": "COST",
            "FilterExpression": {
                "Tags": {"Key": "CostCenter", "Values": ["data-platform"], "MatchOptions": ["EQUALS"]}
            },
        },
        NotificationsWithSubscribers=[
            {
                "Notification": {
                    "NotificationType": "ACTUAL",
                    "ComparisonOperator": "GREATER_THAN",
                    "Threshold": 80.0,
                    "ThresholdType": "PERCENTAGE",
                },
                "Subscribers": [{"SubscriptionType": "EMAIL", "Address": ALERT_EMAIL}],
            },
            {
                "Notification": {
                    "NotificationType": "FORECASTED",
                    "ComparisonOperator": "GREATER_THAN",
                    "Threshold": 100.0,
                    "ThresholdType": "PERCENTAGE",
                },
                "Subscribers": [{"SubscriptionType": "EMAIL", "Address": ALERT_EMAIL}],
            },
        ],
    )
    print(f"Budget {BUDGET_NAME} created")
except budgets.exceptions.DuplicateRecordException:
    print(f"Budget {BUDGET_NAME} already exists")

# 4. Kinesis: alert if records are > 15 minutes old (data freshness problem)
cloudwatch.put_metric_alarm(
    AlarmName="Kinesis-GetRecords-IteratorAge-High",
    MetricName="GetRecords.IteratorAgeMilliseconds",
    Namespace="AWS/Kinesis",
    Dimensions=[{"Name": "StreamName", "Value": STREAM_NAME}],
    Statistic="Maximum",
    Period=300,
    EvaluationPeriods=2,
    Threshold=900_000,   # 15 minutes in milliseconds
    ComparisonOperator="GreaterThanThreshold",
    AlarmDescription="Kinesis consumers are falling behind — data is > 15 min stale",
    AlarmActions=[topic_arn],
)

# 5. Redshift: alert if CPU > 80% for 10 minutes (cluster may need scaling)
cloudwatch.put_metric_alarm(
    AlarmName="Redshift-CPU-High",
    MetricName="CPUUtilization",
    Namespace="AWS/Redshift",
    Dimensions=[{"Name": "ClusterIdentifier", "Value": CLUSTER_ID}],
    Statistic="Average",
    Period=300,
    EvaluationPeriods=2,
    Threshold=80.0,
    ComparisonOperator="GreaterThanThreshold",
    AlarmActions=[topic_arn],
)

print("Budget and alarms configured")
```

**Expected output:** the topic ARN line, the tag line, `Budget cloudmetrics-data-platform created`, `Budget and alarms configured`. Click **Confirm subscription** in the email.

**Test the alert path** (never trust an untested alarm):

```bash
aws cloudwatch set-alarm-state --alarm-name Redshift-CPU-High \
    --state-value ALARM --state-reason "Testing the alert path"
```

**Expected output:** no CLI output; an `ALARM: "Redshift-CPU-High"` email arrives within a minute or two. The alarm returns to its real state at the next evaluation.

---

## PART 6: COST ANALYSIS (30 min)

### Step 6.1: Monthly Cost Comparison

```python
# scripts/11_cost_analysis.py   (needs: python -m pip install pandas tabulate)
import pandas as pd

# Production-scale estimates for the CloudMetrics proposal.
# Prices change — re-check every line in the AWS Pricing Calculator before presenting.
current = [
    ("RDS PostgreSQL r6g.2xlarge (2TB GP3)",     2800, "Cannot scale past 2TB"),
    ("RDS Backups (2TB, 30-day retention)",        400, "Point-in-time recovery"),
]
proposed = [
    # S3
    ("S3 Standard (500GB new raw/month)",           12, "~500GB/month new data"),
    ("S3 Standard-IA (1TB processed)",              13, "90+ day data auto-tiered"),
    ("S3 Glacier IR (2TB historical > 1yr)",         8, "Long-tail archive"),
    ("S3 API Requests (10M PUT, 50M GET)",          25, "Firehose writes, Glue/Spectrum reads"),
    # Kinesis
    ("Kinesis ON_DEMAND (200M events/day)",        450, "Auto-scales, ~$0.08/GB"),
    ("Kinesis Firehose (6TB/month delivered)",     210, "$0.035/GB delivered to S3"),
    # Redshift
    ("Redshift ra3.xlplus x3 (1-year RI)",         662, "Reserved pricing — verify in the calculator"),
    ("Redshift Spectrum (2TB scanned/month)",       10, "$5/TB, Parquet minimises scan"),
    # Glue
    ("Glue ETL (10 DPU × 2hr/day)",                264, "10 DPU × $0.44 × 2hr × 30 days"),
    # Other
    ("CloudWatch monitoring + alarms",              50, "Custom metrics and dashboards"),
    ("Data Transfer (VPC endpoints = free for S3)", 20, "S3 VPC endpoint avoids NAT cost"),
]

current_total = sum(cost for _, cost, _ in current)
proposed_total = sum(cost for _, cost, _ in proposed)

rows = (current + [("CURRENT TOTAL", current_total, "No path to 50TB"), ("", None, "")]
        + proposed + [("PROPOSED TOTAL", proposed_total, "Scales to 50TB+ with minimal cost growth")])

df = pd.DataFrame(rows, columns=["Component", "Monthly Cost (USD)", "Notes"])
print(df.to_markdown(index=False))   # requires the tabulate package

savings = current_total - proposed_total
print("\n" + "=" * 60)
print(f"Savings vs current state: ${savings:,}/month (${savings * 12:,}/year)")
print("Scale ceiling: Current 2TB vs Proposed UNLIMITED")
print("=" * 60)
```

**Expected output** (end of the printout):

```
Savings vs current state: $1,476/month ($17,712/year)
Scale ceiling: Current 2TB vs Proposed UNLIMITED
```

---

## PART 7: TEARDOWN (mandatory — 20 min)

Delete everything the same day you finish. The script works in dependency order and is safe to re-run (things already gone are skipped). It **schedules** the KMS key for deletion (7-day minimum waiting period — the key stops billing immediately) and keeps your learner budgets from granular Lab_00.

```python
# scripts/99_teardown.py — delete every billable resource from this lab
import boto3
from botocore.exceptions import ClientError, WaiterError

from cm_config import (BUCKET, BUDGET_NAME, CLUSTER_ID, FIREHOSE_STACK, GLUE_DATABASE, KMS_ALIAS,
                       REGION, ROLE_FIREHOSE, ROLE_GLUE, ROLE_PRODUCER, ROLE_REDSHIFT,
                       SECURITY_GROUP, SNS_TOPIC, STREAM_NAME, SUBNET_GROUP, account_id)

ACCOUNT = account_id()


def step(label, fn):
    try:
        fn()
        print(f"[deleted] {label}")
    except ClientError as exc:
        print(f"[skipped] {label}: {exc.response['Error']['Code']}")
    except WaiterError as exc:
        print(f"[CHECK]   {label}: still deleting or failed ({exc}) — re-run this script in a few minutes")


cfn = boto3.client("cloudformation", region_name=REGION)
kinesis = boto3.client("kinesis", region_name=REGION)
redshift = boto3.client("redshift", region_name=REGION)
ec2 = boto3.client("ec2", region_name=REGION)
glue = boto3.client("glue", region_name=REGION)
s3 = boto3.resource("s3", region_name=REGION)
kms = boto3.client("kms", region_name=REGION)
iam = boto3.client("iam")
sns = boto3.client("sns", region_name=REGION)
cloudwatch = boto3.client("cloudwatch", region_name=REGION)
budgets = boto3.client("budgets", region_name="us-east-1")

# 1. Firehose (CloudFormation stack) — stop delivery first
step("Firehose stack", lambda: (cfn.delete_stack(StackName=FIREHOSE_STACK),
                                cfn.get_waiter("stack_delete_complete").wait(StackName=FIREHOSE_STACK)))
# 2. Kinesis stream
step("Kinesis stream", lambda: kinesis.delete_stream(StreamName=STREAM_NAME, EnforceConsumerDeletion=True))
# 3. Redshift cluster (no final snapshot — this is a lab), then its network pieces
step("Redshift cluster", lambda: (
    redshift.delete_cluster(ClusterIdentifier=CLUSTER_ID, SkipFinalClusterSnapshot=True),
    redshift.get_waiter("cluster_deleted").wait(ClusterIdentifier=CLUSTER_ID,
                                                WaiterConfig={"Delay": 30, "MaxAttempts": 60})))
step("Cluster subnet group", lambda: redshift.delete_cluster_subnet_group(ClusterSubnetGroupName=SUBNET_GROUP))
for sg in ec2.describe_security_groups(Filters=[{"Name": "group-name", "Values": [SECURITY_GROUP]}])["SecurityGroups"]:
    step(f"Security group {sg['GroupId']}", lambda sg=sg: ec2.delete_security_group(GroupId=sg["GroupId"]))
# 4. Glue Data Catalog (deleting the database deletes its tables and partitions)
step("Glue database", lambda: glue.delete_database(Name=GLUE_DATABASE))
# 5. S3: delete every object version and delete marker, then the bucket
bucket = s3.Bucket(BUCKET)
step("S3 objects (all versions)", lambda: bucket.object_versions.delete())
step("S3 bucket", lambda: bucket.delete())
# 6. KMS: remove the alias and schedule the key for deletion (7-day minimum)
def delete_key():
    key_id = kms.describe_key(KeyId=KMS_ALIAS)["KeyMetadata"]["KeyId"]
    kms.delete_alias(AliasName=KMS_ALIAS)
    kms.schedule_key_deletion(KeyId=key_id, PendingWindowInDays=7)
step("KMS key (scheduled, 7 days)", delete_key)
# 7. IAM roles: inline policies first, then the role
for role in (ROLE_PRODUCER, ROLE_FIREHOSE, ROLE_GLUE, ROLE_REDSHIFT):
    def delete_role(role=role):
        for name in iam.list_role_policies(RoleName=role)["PolicyNames"]:
            iam.delete_role_policy(RoleName=role, PolicyName=name)
        iam.delete_role(RoleName=role)
    step(f"IAM role {role}", delete_role)
# 8. Monitoring and cost controls
step("CloudWatch alarms", lambda: cloudwatch.delete_alarms(
    AlarmNames=["Kinesis-GetRecords-IteratorAge-High", "Redshift-CPU-High"]))
step("SNS topic", lambda: sns.delete_topic(TopicArn=f"arn:aws:sns:{REGION}:{ACCOUNT}:{SNS_TOPIC}"))
step("Budget", lambda: budgets.delete_budget(AccountId=ACCOUNT, BudgetName=BUDGET_NAME))

print("Teardown finished — verify with the commands below.")
```

**Expected output:** a `[deleted]` line per resource (the cluster wait takes ~5–10 minutes). `[skipped] … ResourceNotFoundException` / `NoSuchBucket` lines are fine — that thing was already gone.

**Verify** (Git Bash):

```bash
aws kinesis list-streams --query 'StreamNames'                          # []
aws firehose list-delivery-streams --query 'DeliveryStreamNames'        # []
aws redshift describe-clusters --query 'Clusters[].ClusterIdentifier'   # []
aws s3 ls | grep cloudmetrics || echo "no cloudmetrics buckets"
aws resourcegroupstaggingapi get-resources --region us-east-1 \
    --tag-filters Key=CostCenter,Values=data-platform \
    --query 'ResourceTagMappingList[].ResourceARN'
```

**Expected output:** empty lists and `no cloudmetrics buckets`. The tagging API can list a deleted resource for a few hours, and the KMS key shows as `PendingDeletion` for 7 days — both are fine. Check Billing → Bills tomorrow.

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The Kinesis stream uses `customer_id` as the partition key. A new enterprise customer signs up with 10x the API call volume of any existing customer. What happens to the Kinesis stream's shard distribution, and how would you detect and fix it?

2. The S3 lifecycle policy moves raw data to STANDARD\_IA after 30 days. An analyst wants to query raw files from 45 days ago using Athena. What happens, and is there any cost implication? Would the behaviour be different for files that have reached Glacier Instant Retrieval or Deep Archive?

3. The Redshift Spectrum role has `s3:GetObject` on the entire data lake bucket. The security team flags this as overly permissive. Propose a more tightly scoped permission that still allows Spectrum to query the curated zone without exposing the raw zone.

4. The cost analysis shows the proposed architecture at $1,724/month vs $3,200/month today. The CTO asks: "What happens to the cost if customer count grows from 5,000 to 50,000?" Walk through which line items scale linearly with data volume and which are roughly fixed.

5. You need to re-run the daily COPY load for March 19 because the Glue job wrote corrupted Parquet files. The manifest for March 19 still points to the old files. Walk through the exact sequence of steps to: produce corrected files, update the manifest, re-run the COPY, and verify the result is correct.
