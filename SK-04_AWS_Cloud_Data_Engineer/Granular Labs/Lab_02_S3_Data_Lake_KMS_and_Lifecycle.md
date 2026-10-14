# Lab 02 — S3 Data Lake with KMS Encryption and Lifecycle

> **Goal:** Build the CloudMetrics data lake bucket exactly as the guided lab does — zone prefixes, Block Public Access, versioning, SSE-KMS with a customer managed key and Bucket Key, tags, and a multi-tier lifecycle policy.
> **Time:** 90–120 min | **Difficulty:** Beginner | **Prerequisites:** Lab 01
> **Cost:** < $0.10 — S3 storage is pennies; the KMS key costs $1/month (~$0.03/day) until scheduled for deletion. Keep the bucket and key for Labs 03–06 (Teardown explains the full delete).
> **Guided lab coverage:** PART 1 (Step 1.1 bucket + KMS, Step 1.2 lifecycle)

## Learning objectives

By the end of this lab you can:

- Explain buckets, keys, prefixes, and the raw / processed / curated / rejected / temp zone layout.
- Secure a bucket with Block Public Access, versioning and default **SSE-KMS** using a **customer managed key (CMK)** with **Bucket Key** enabled.
- Capture command output in bash variables (the KMS key ID) and write idempotent scripts.
- Design a lifecycle policy (Standard → Standard-IA → Glacier Instant Retrieval → Deep Archive → expire) and explain its cost trade-offs.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Understand the lake layout

S3 stores **objects** in **buckets**; each object has a **key** such as `raw/events/year=2026/month=03/day=19/hour=14/file.parquet`. There are no real folders — `/` is just a character the console draws as folders. CloudMetrics uses **one bucket** with zone **prefixes**:

| Prefix | Content | Who writes it |
|---|---|---|
| `raw/` | Events as delivered (Firehose converts JSON → Parquet) | Firehose (Lab 04) |
| `processed/` | Cleaned Parquet, partitioned by year/month/day | Future Glue jobs |
| `curated/` | Star-schema data for Spectrum | Future Glue jobs |
| `rejected/` | Quality failures and Firehose errors | Firehose / ETL |
| `manifests/` | COPY manifests | Lab 06 load script |
| `temp/` | Scratch space, auto-deleted after 7 days | Anyone |

Keys like `year=2026/month=03/` are **Hive-style partitions**: engines (Spectrum, Athena, Glue) read `column=value` segments as columns and skip folders that do not match a filter.

**Expected output:** none — this step is reading.

✅ **Checkpoint:** you can say which prefix each service writes to and why `raw/` must never be modified by ETL (it is the replay source).

### Step 2 — Create, tag and lock the bucket

```bash
# us-east-1 is the one region where you must NOT pass --create-bucket-configuration
aws s3api create-bucket --bucket "$BUCKET" --region "$REGION"

aws s3api put-bucket-tagging --bucket "$BUCKET" \
    --tagging 'TagSet=[{Key=CostCenter,Value=data-platform},{Key=Environment,Value=learning}]'

aws s3api put-public-access-block --bucket "$BUCKET" \
    --public-access-block-configuration \
        "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

aws s3api put-bucket-versioning --bucket "$BUCKET" --versioning-configuration Status=Enabled

aws s3api get-public-access-block --bucket "$BUCKET"
aws s3api get-bucket-versioning --bucket "$BUCKET"
```

**Expected output:**

```json
{
    "Location": "/cloudmetrics-data-lake-francis"
}
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
{
    "Status": "Enabled"
}
```

✅ **Checkpoint:** all four public-access flags are `true` and versioning is `Enabled`. The `CostCenter` tag is how Lab 07's budget finds this bucket's cost.

### Step 3 — Create a customer managed KMS key (and capture its ID)

**SSE-S3** (AWS-managed keys) is free but gives no control. A **customer managed key (CMK)** in KMS costs $1/month and gives you key rotation, a key policy, and a CloudTrail record of every decrypt — what compliance teams (GDPR/SOC2 for CloudMetrics) expect. An **alias** (`alias/cloudmetrics-data-key`) is a friendly name pointing at the key.

The key ID is only printed once, so **capture it in a variable** with `$( … )` and `--query … --output text`:

```bash
KEY_ID=$(aws kms create-key \
    --description "CloudMetrics data lake encryption key" \
    --tags TagKey=CostCenter,TagValue=data-platform \
    --region "$REGION" \
    --query 'KeyMetadata.KeyId' --output text)
echo "KEY_ID=$KEY_ID"

aws kms create-alias --alias-name "$KMS_ALIAS" --target-key-id "$KEY_ID" --region "$REGION"
aws kms enable-key-rotation --key-id "$KEY_ID" --region "$REGION"
aws kms describe-key --key-id "$KMS_ALIAS" --region "$REGION" \
    --query 'KeyMetadata.[KeyId,KeyState,KeyManager]'
```

**Expected output:**

```
KEY_ID=1234abcd-12ab-34cd-56ef-1234567890ab
[
    "1234abcd-12ab-34cd-56ef-1234567890ab",
    "Enabled",
    "CUSTOMER"
]
```

✅ **Checkpoint:** the alias resolves to your key and `KeyManager` is `CUSTOMER`. Run `create-key` only **once** — every extra key costs $1/month. (The full script in Step 6 checks for the alias first, so re-running it never creates a second key.)

### Step 4 — Default encryption with SSE-KMS and Bucket Key

**Bucket Key** makes S3 reuse a short-lived bucket-level data key instead of calling KMS for every object — up to 99% fewer KMS requests (and cost).

```bash
KEY_ARN=$(aws kms describe-key --key-id "$KMS_ALIAS" --region "$REGION" --query 'KeyMetadata.Arn' --output text)

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

echo "hello lake" > hello.txt
aws s3 cp hello.txt "s3://$BUCKET/temp/hello.txt"
aws s3api head-object --bucket "$BUCKET" --key temp/hello.txt \
    --query '{SSE:ServerSideEncryption,Key:SSEKMSKeyId,BucketKey:BucketKeyEnabled}'
```

The quoting `'…"'"$KEY_ARN"'"…'` closes the single-quoted JSON, inserts the variable inside double quotes, and reopens it — the same trick the guided lab uses.

**Expected output:**

```
upload: ./hello.txt to s3://cloudmetrics-data-lake-francis/temp/hello.txt
{
    "SSE": "aws:kms",
    "Key": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
    "BucketKey": true
}
```

✅ **Checkpoint:** you uploaded with no encryption flags, yet the object is `aws:kms` with **your** key — default encryption works. Every service that writes or reads this bucket now needs KMS permissions (that is why the guided lab's roles contain `kms:` statements).

### Step 5 — See versioning protect you

```bash
echo "hello lake v2" > hello.txt
aws s3 cp hello.txt "s3://$BUCKET/temp/hello.txt"
aws s3api list-object-versions --bucket "$BUCKET" --prefix temp/hello.txt \
    --query 'Versions[].{VersionId:VersionId,IsLatest:IsLatest}'
```

**Expected output:**

```json
[
    {
        "VersionId": "3HL4kqtJlcpXroDTDmJ.rmSpXd3dIbrHY",
        "IsLatest": true
    },
    {
        "VersionId": "UIORUnfndfhnw89493jJFJ.UmJSDfGGh",
        "IsLatest": false
    }
]
```

✅ **Checkpoint:** two versions — the overwrite did not destroy the original. Remember: old versions bill too, and a versioned bucket must have **every version** deleted before it can be removed (Lab 07's teardown script does this).

### Step 6 — Put it together: `scripts/01_create_lake.sh`

Turn Steps 2–4 into the guided lab's re-runnable script. Note `set -euo pipefail` (stop on the first error) and the `if KEY_ID=$(…describe-key…)` test that **reuses** the key when the alias exists:

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

```bash
bash scripts/01_create_lake.sh
```

**Expected output** (the bucket and key already exist from Steps 2–4):

```
Reusing existing key 1234abcd-12ab-34cd-56ef-1234567890ab (alias/cloudmetrics-data-key)
S3 bucket cloudmetrics-data-lake-francis configured successfully
```

(`create-bucket` on a bucket you already own in us-east-1 succeeds silently and prints the `Location` again.)

✅ **Checkpoint:** the script runs twice without errors and without creating a second key — that is idempotency.

### Step 7 — Lifecycle policy with boto3: `scripts/02_lifecycle.py`

Storage classes trade storage price against retrieval cost and minimum storage duration:

| Class | ~Price/TB-month | Minimum duration | Use for |
|---|---|---|---|
| S3 Standard | $23 | none | hot data (first 30 days) |
| Standard-IA | $12.50 | 30 days, per-GB retrieval fee | occasionally read |
| Glacier Instant Retrieval | $4 | 90 days, higher retrieval fee | rarely read, still milliseconds |
| Deep Archive | $0.99 | 180 days, hours to restore | compliance archive |

Save and run the guided lab's lifecycle script:

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

```bash
python scripts/02_lifecycle.py
aws s3api get-bucket-lifecycle-configuration --bucket "$BUCKET" --query 'Rules[].ID'
```

**Expected output:**

```
Lifecycle policy applied
[
    "RawZone-CostTiering",
    "ProcessedZone-CostTiering",
    "TempZone-AutoDelete",
    "CleanupIncompleteMultipartUploads"
]
```

✅ **Checkpoint:** you can explain each rule: raw ages through IA → Glacier IR → Deep Archive and expires after 7 years (2,555 days, the compliance requirement); `temp/` self-cleans after 7 days; abandoned multipart uploads stop billing after 7 days.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `BucketAlreadyExists` | Another AWS customer owns that name | Change `NAME` in `scripts/cm_config.py` (e.g. add a digit) and re-run `eval "$(python scripts/cm_config.py)"` |
| `InvalidLocationConstraint` | `--create-bucket-configuration LocationConstraint=us-east-1` was passed | Omit it in us-east-1 |
| `AlreadyExistsException` on `create-alias` | The alias already points to a key | Reuse it: `aws kms describe-key --key-id "$KMS_ALIAS"` — do **not** create another key |
| `KMS.NotFoundException` / `KMSInvalidStateException` | The key is pending deletion (from an earlier teardown) | Delete the alias (`aws kms delete-alias --alias-name "$KMS_ALIAS"`) and run `bash scripts/01_create_lake.sh` to create a fresh key |
| `MalformedXML` from lifecycle | Invalid rule (e.g. IA transition < 30 days) | Compare with the script; transitions must respect minimum durations |
| `$BUCKET` is empty | You skipped the session commands | Run `eval "$(python scripts/cm_config.py)"` |

## Teardown

**Keep the bucket and key** if you continue to Lab 03 within the next week — idle cost is a few cents (key $1/month prorated, a few KB of storage). Remove only the practice file: `rm hello.txt`.

**Stopping SK-04 for more than a week?** Delete everything (every object version, the bucket, the alias, and schedule the key for deletion — 7-day minimum wait; a pending key does not bill):

```bash
python - <<'EOF'
import sys
sys.path.insert(0, "scripts")
import boto3
from cm_config import BUCKET, KMS_ALIAS, REGION

bucket = boto3.resource("s3", region_name=REGION).Bucket(BUCKET)
bucket.object_versions.delete()
bucket.delete()
kms = boto3.client("kms", region_name=REGION)
key_id = kms.describe_key(KeyId=KMS_ALIAS)["KeyMetadata"]["KeyId"]
kms.delete_alias(AliasName=KMS_ALIAS)
kms.schedule_key_deletion(KeyId=key_id, PendingWindowInDays=7)
print("Bucket deleted; key scheduled for deletion")
EOF
```

**Expected output:** `Bucket deleted; key scheduled for deletion`. Rebuild later with `bash scripts/01_create_lake.sh && python scripts/02_lifecycle.py`.

## Key takeaways

- One bucket, zone prefixes, Hive-style partition keys.
- Secure at creation: Block Public Access, versioning, SSE-KMS with a CMK and Bucket Key, tags.
- Capture CLI output with `$(… --query … --output text)`; make scripts idempotent.
- Lifecycle rules turn storage cost into policy — mind minimum durations and retrieval fees.

## Next

[Lab 03 — Kinesis Data Streams and a Resilient Producer](Lab_03_Kinesis_Data_Streams_and_Producer.md)
