# Lab 04 — Glue Data Catalog, CloudFormation and Firehose

> **Goal:** Deliver Kinesis events into the lake as partitioned, KMS-encrypted Parquet using a Glue Data Catalog schema, the guided lab's IAM roles, and a Firehose delivery stream deployed with CloudFormation.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 03 (and the Lab 02 bucket + KMS key still in place)
> **Cost:** ~$0.20 — the Kinesis stream bills ~$0.04/hour; Firehose bills per GB (+ Parquet conversion), pennies here. **Delete the Firehose stack and the stream in Teardown.** Keep the bucket data, Glue table and IAM roles for Lab 06.
> **Guided lab coverage:** PART 2 (Step 2.1 — all four service roles), PART 3 — Step 3.2 (Glue Data Catalog table), Step 3.3 (Firehose CloudFormation), Step 3.4 (producer → S3)

## Learning objectives

By the end of this lab you can:

- Read and write a CloudFormation template (Parameters, Resources, Outputs, `Ref`, `Fn::Sub`, `Fn::GetAtt`, pseudo parameters) and deploy, inspect and delete a stack.
- Explain what the Glue Data Catalog is and create a partitioned Parquet table with boto3.
- Create the guided lab's four least-privilege service roles.
- Configure Firehose buffering, JSON → Parquet conversion, timestamp-based prefixes and an error prefix — and see both the success and the error path.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — CloudFormation fundamentals with a practice stack

**CloudFormation** creates AWS resources from a template (JSON or YAML). You deploy a template as a **stack**; deleting the stack deletes everything it created. Template anatomy:

| Section | Purpose |
|---|---|
| `Parameters` | Inputs supplied at deploy time (`--parameter-overrides Name=value`) |
| `Resources` | What to create; each has a logical name, a `Type` and `Properties` |
| `Outputs` | Values to show after deployment (ARNs, names) |
| `{"Ref": "X"}` | Value of parameter X, or the default ID of resource X |
| `{"Fn::Sub": "…${X}…"}` | String with parameters / pseudo parameters substituted |
| `{"Fn::GetAtt": ["Res", "Arn"]}` | An attribute of a resource |
| `AWS::Region`, `AWS::AccountId` | Pseudo parameters — filled in by CloudFormation |

Save `infra/lab04_practice.json`:

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Description": "Lab 04 practice stack - one SNS topic",
    "Parameters": {
        "TopicSuffix": {"Type": "String", "Default": "practice"}
    },
    "Resources": {
        "PracticeTopic": {
            "Type": "AWS::SNS::Topic",
            "Properties": {
                "TopicName": {"Fn::Sub": "cloudmetrics-cfn-${TopicSuffix}"},
                "Tags": [{"Key": "CostCenter", "Value": "data-platform"}]
            }
        }
    },
    "Outputs": {
        "TopicArn":     {"Value": {"Ref": "PracticeTopic"}},
        "DeployRegion": {"Value": {"Ref": "AWS::Region"}}
    }
}
```

```bash
aws cloudformation validate-template --template-body file://infra/lab04_practice.json --query Parameters
aws cloudformation deploy --template-file infra/lab04_practice.json --stack-name cloudmetrics-cfn-practice
aws cloudformation describe-stacks --stack-name cloudmetrics-cfn-practice --query 'Stacks[0].Outputs'
```

**Expected output:**

```
[
    {
        "ParameterKey": "TopicSuffix",
        "DefaultValue": "practice",
        "NoEcho": false
    }
]

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - cloudmetrics-cfn-practice
[
    {
        "OutputKey": "TopicArn",
        "OutputValue": "arn:aws:sns:us-east-1:123456789012:cloudmetrics-cfn-practice"
    },
    {
        "OutputKey": "DeployRegion",
        "OutputValue": "us-east-1"
    }
]
```

Now delete the stack and wait until it is gone:

```bash
aws cloudformation delete-stack --stack-name cloudmetrics-cfn-practice
aws cloudformation wait stack-delete-complete --stack-name cloudmetrics-cfn-practice
aws sns list-topics --query 'Topics[].TopicArn'
```

**Expected output:** the wait returns silently; the topic list no longer contains `cloudmetrics-cfn-practice`.

✅ **Checkpoint:** you can explain what `deploy` did (created a **change set**, then executed it) and why deleting a stack is a reliable teardown. (Terraform, which you may know from elsewhere, does the same job with `.tf` files and `terraform destroy`; the guided lab uses CloudFormation.)

### Step 2 — Recreate the stream

Firehose reads from the stream, so it must exist first:

```bash
python scripts/04_create_stream.py
```

**Expected output:** `Stream cloudmetrics-events is ACTIVE` (the hourly charge restarts now).

✅ **Checkpoint:** `aws kinesis list-streams --query StreamNames` shows `["cloudmetrics-events"]`.

### Step 3 — Register the schema in the Glue Data Catalog: `scripts/05_glue_catalog.py`

Files in S3 are just bytes. The **Glue Data Catalog** stores *table definitions* — columns, types, file format, S3 location, partition keys — that many engines share. In CloudMetrics, **Firehose** reads the table to convert JSON to Parquet, and **Redshift Spectrum** reads it to query S3 (Lab 06). The partition keys (`year/month/day/hour`) match the Firehose prefix you will configure in Step 5.

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

```bash
python scripts/05_glue_catalog.py
aws glue get-table --database-name "$GLUE_DATABASE" --name "$GLUE_TABLE" \
    --query 'Table.{Columns:StorageDescriptor.Columns[].Name,Partitions:PartitionKeys[].Name,Location:StorageDescriptor.Location}'
```

**Expected output:**

```
Created table cloudmetrics_raw.events
{
    "Columns": ["event_id", "customer_id", "event_type", "endpoint", "method", "status_code",
                "response_time_ms", "request_size_bytes", "response_size_bytes", "event_timestamp", "region"],
    "Partitions": ["year", "month", "day", "hour"],
    "Location": "s3://cloudmetrics-data-lake-francis/raw/events/"
}
```

✅ **Checkpoint:** the column names match the JSON keys the producer sends (`event_timestamp`, not `timestamp`) — Firehose matches JSON keys to columns **by name**.

### Step 4 — Create the four service roles: `scripts/03_iam_roles.py`

You built Role 1 by hand in Lab 01. This is the guided lab's complete IAM model — read each policy and match it to the table you filled in `notes/iam.md`. It needs the KMS key from Lab 02.

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

```bash
python scripts/03_iam_roles.py
aws iam list-roles --query "Roles[?starts_with(RoleName, 'cloudmetrics-')].RoleName"
```

**Expected output:**

```
Created cloudmetrics-kinesis-producer-role
Created cloudmetrics-firehose-delivery-role
Created cloudmetrics-glue-etl-role
Created cloudmetrics-redshift-spectrum-role
All service roles created
[
    "cloudmetrics-firehose-delivery-role",
    "cloudmetrics-glue-etl-role",
    "cloudmetrics-kinesis-producer-role",
    "cloudmetrics-redshift-spectrum-role"
]
```

✅ **Checkpoint:** you can point to the statements that let Firehose (a) read the stream, (b) write only `raw/` and `rejected/firehose/`, (c) read the Glue table, (d) use the key only through S3.

### Step 5 — Firehose with CloudFormation: `infra/firehose.json`

Firehose design choices in the template:

| Setting | Value | Why |
|---|---|---|
| `DeliveryStreamType` | `KinesisStreamAsSource` | Firehose consumes the Kinesis stream (it starts reading at the moment it is created) |
| `BufferingHints` | 128 MB **or** 300 s | Few large files instead of millions of tiny ones; Parquet conversion requires ≥ 64 MB |
| `Prefix` | `raw/events/year=!{timestamp:yyyy}/…/hour=!{timestamp:HH}/` | Hive-style partitions from the record's **arrival time (UTC)** — matches the Glue partition keys |
| `ErrorOutputPrefix` | `rejected/firehose/!{firehose:error-output-type}/…` | Failed records land in `rejected/`, split by error type, instead of being lost |
| `DataFormatConversionConfiguration` | OpenX JSON in → Parquet (Snappy) out, schema from Glue | Columnar, compressed files for Redshift and Spectrum |
| `RoleARN` (3 places) | `cloudmetrics-firehose-delivery-role` | One role for source, destination and schema — the role from Step 4 |

Save the guided lab's template as `infra/firehose.json`:

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

Validate and deploy:

```bash
aws cloudformation validate-template --template-body file://infra/firehose.json --query 'Parameters[].ParameterKey'
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

**Expected output:**

```
[
    "BucketName",
    "FirehoseRoleArn",
    "StreamName",
    "GlueDatabase",
    "GlueTable"
]

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - cloudmetrics-firehose
"ACTIVE"
```

✅ **Checkpoint:** the delivery stream is `ACTIVE`. If the stack failed, run `aws cloudformation describe-stack-events --stack-name cloudmetrics-firehose --max-items 5` — the first `CREATE_FAILED` event says why (see Troubleshooting).

### Step 6 — Send events and watch Parquet land in the lake

```bash
python scripts/06_producer.py
echo "Waiting 6 minutes for the Firehose buffer (300 s) to flush..."; sleep 360
aws s3 ls "s3://$BUCKET/raw/events/" --recursive
```

**Expected output:**

```
Sent: 10,000 | Failed: 0
Waiting 6 minutes for the Firehose buffer (300 s) to flush...
2026-03-19 14:10:12     201234 raw/events/year=2026/month=03/day=19/hour=14/cloudmetrics-events-to-s3-1-2026-03-19-14-05-12-1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d
```

Check one object's encryption (copy the key from your listing):

```bash
KEY=$(aws s3api list-objects-v2 --bucket "$BUCKET" --prefix raw/events/ --query 'Contents[0].Key' --output text)
aws s3api head-object --bucket "$BUCKET" --key "$KEY" --query '{SSE:ServerSideEncryption,BucketKey:BucketKeyEnabled,Size:ContentLength}'
```

**Expected output:**

```json
{
    "SSE": "aws:kms",
    "BucketKey": true,
    "Size": 201234
}
```

✅ **Checkpoint:** one or a few objects under `year=/month=/day=/hour=` — 10,000 JSON events became a small number of compressed Parquet files, encrypted with your key. Note the date: Lab 06 loads this day.

### Step 7 — See the error path

Send a record that is not JSON at all:

```bash
aws kinesis put-record --stream-name "$STREAM_NAME" --partition-key bad-0001 \
    --cli-binary-format raw-in-base64-out --data 'this is not json'
echo "Waiting 6 minutes..."; sleep 360
aws s3 ls "s3://$BUCKET/rejected/" --recursive
```

**Expected output:**

```
2026-03-19 14:22:40       1234 rejected/firehose/format-conversion-failed/year=2026/month=03/day=19/cloudmetrics-events-to-s3-1-2026-03-19-14-17-35-...
```

✅ **Checkpoint:** the bad record was **quarantined, not lost** — that is what `ErrorOutputPrefix` is for. (`aws s3 cp s3://$BUCKET/<that key> -` shows a JSON line with the error and the base64 raw data.)

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Stack `CREATE_FAILED` … "not authorized to perform: kinesis:DescribeStream" / "unable to assume role" | Role missing or too new (IAM propagation) | Run `python scripts/03_iam_roles.py`, wait 30 s, delete the failed stack (`aws cloudformation delete-stack …`) and deploy again |
| Stack fails with "Stream … not found" | Kinesis stream not created / deleted | `python scripts/04_create_stream.py`, then redeploy |
| `ROLLBACK_COMPLETE` and `deploy` refuses to update | A stack that failed on creation cannot be updated | `aws cloudformation delete-stack --stack-name cloudmetrics-firehose`, wait, redeploy |
| No objects after 6 minutes | Producer ran before Firehose existed, or Firehose has no S3/KMS permission | Re-run the producer; check `rejected/` and the console → Firehose → stream → **Destination error logs** |
| Objects under `rejected/firehose/format-conversion-failed/` for real events | JSON field names/types do not match the Glue table | Compare producer keys with Step 3 columns (`event_timestamp`!) |
| `KMS.AccessDeniedException` in Firehose errors | Key ARN in the role policy does not match the bucket's key | Re-run `python scripts/03_iam_roles.py` (it reads the current key ARN) |

## Teardown

**Mandatory today** — delete the delivery stream first (it reads from the stream), then the stream:

```bash
aws cloudformation delete-stack --stack-name cloudmetrics-firehose
aws cloudformation wait stack-delete-complete --stack-name cloudmetrics-firehose
aws kinesis delete-stream --stream-name "$STREAM_NAME"
aws firehose list-delivery-streams --query DeliveryStreamNames
aws kinesis list-streams --query StreamNames
```

**Expected output:** `[]` and `[]` (the stream can take ~30 s to disappear).

**Keep** the bucket (with the Parquet files), the KMS key, the Glue table and the four IAM roles — Lab 06 loads these files into Redshift. Idle cost: cents.

## Key takeaways

- CloudFormation = templates → stacks; parameters make one template work for every trainee and environment; deleting the stack is the teardown.
- The Glue Data Catalog is the shared schema: Firehose uses it to write Parquet, Spectrum uses it to read.
- Firehose buffering decides file size; timestamp prefixes create Hive partitions; the error prefix keeps bad data instead of dropping it.
- Order matters: role → stream → Glue table → Firehose → producer.

## Next

[Lab 05 — Redshift Cluster, Networking and Table Design](Lab_05_Redshift_Cluster_and_Table_Design.md)
