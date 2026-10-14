# Lab 01 — AWS Foundations, IAM and boto3

> **Goal:** Understand regions, ARNs and IAM policies well enough to create a least-privilege service role with boto3 and prove what it can and cannot do.
> **Time:** 90–120 min | **Difficulty:** Beginner | **Prerequisites:** Lab 00
> **Cost:** $0.00 — IAM and the policy simulator are free. Delete the practice role in Teardown.
> **Guided lab coverage:** PART 2 (Step 2.1 — `create_role`, trust policies, inline policies, `kms:ViaService` conditions) and the ARNs used throughout Parts 1–5

## Learning objectives

By the end of this lab you can:

- Explain regions and Availability Zones, and read any ARN.
- Explain users vs roles, and **trust policy** (who may assume a role) vs **permission policy** (what the role may do).
- Create a role with boto3 (`create_role` + `put_role_policy`) that is safe to re-run.
- Use the IAM policy simulator API to prove a policy allows exactly what it should — including bucket-vs-object ARNs and condition keys.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Regions and Availability Zones

A **region** is a cluster of data centres in one geography; resources live in exactly one region. An **Availability Zone (AZ)** is one or more independent data centres inside a region. CloudMetrics (and you) use `us-east-1`.

```bash
aws ec2 describe-availability-zones --region us-east-1 --query 'AvailabilityZones[].ZoneName'
```

**Expected output:**

```json
[
    "us-east-1a",
    "us-east-1b",
    "us-east-1c",
    "us-east-1d",
    "us-east-1e",
    "us-east-1f"
]
```

✅ **Checkpoint:** six AZs. In the console, set the region selector (top right) to **US East (N. Virginia)** — "my resource disappeared" is almost always the wrong region selected.

### Step 2 — Read ARNs

An **ARN** (Amazon Resource Name) uniquely identifies a resource: `arn:partition:service:region:account-id:resource`. IAM policies are written in ARNs, so you must be able to build them. Save and run `scripts/lab01_arns.py`:

```python
# scripts/lab01_arns.py — print the ARNs of the resources SK-04 builds
from cm_config import (BUCKET, GLUE_DATABASE, GLUE_TABLE, KMS_ALIAS, REGION, ROLE_FIREHOSE,
                       STREAM_NAME, account_id)

ACCOUNT = account_id()
arns = {
    "S3 bucket":           f"arn:aws:s3:::{BUCKET}",
    "S3 objects in raw/":  f"arn:aws:s3:::{BUCKET}/raw/*",
    "Kinesis stream":      f"arn:aws:kinesis:{REGION}:{ACCOUNT}:stream/{STREAM_NAME}",
    "IAM role":            f"arn:aws:iam::{ACCOUNT}:role/{ROLE_FIREHOSE}",
    "Glue table":          f"arn:aws:glue:{REGION}:{ACCOUNT}:table/{GLUE_DATABASE}/{GLUE_TABLE}",
    "KMS alias":           f"arn:aws:kms:{REGION}:{ACCOUNT}:{KMS_ALIAS}",
}
for name, arn in arns.items():
    print(f"{name:20} {arn}")
```

```bash
python scripts/lab01_arns.py
```

**Expected output** (your name and account differ):

```
S3 bucket            arn:aws:s3:::cloudmetrics-data-lake-francis
S3 objects in raw/   arn:aws:s3:::cloudmetrics-data-lake-francis/raw/*
Kinesis stream       arn:aws:kinesis:us-east-1:123456789012:stream/cloudmetrics-events
IAM role             arn:aws:iam::123456789012:role/cloudmetrics-firehose-delivery-role
Glue table           arn:aws:glue:us-east-1:123456789012:table/cloudmetrics_raw/events
KMS alias            arn:aws:kms:us-east-1:123456789012:alias/cloudmetrics-data-key
```

✅ **Checkpoint:** you can explain why the S3 ARNs have no region or account (bucket names are globally unique) and why IAM ARNs have no region (IAM is global).

### Step 3 — Read an AWS managed policy

A policy is JSON: `Version` (always `2012-10-17`), and `Statement`s with `Effect` (Allow/Deny), `Action` (`service:Operation`, wildcards allowed), `Resource` (ARNs) and optional `Condition`.

```bash
POLICY=arn:aws:iam::aws:policy/AmazonKinesisReadOnlyAccess
VERSION=$(aws iam get-policy --policy-arn "$POLICY" --query Policy.DefaultVersionId --output text)
aws iam get-policy-version --policy-arn "$POLICY" --version-id "$VERSION" --query PolicyVersion.Document
```

**Expected output** (shape — AWS updates managed policies over time):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["kinesis:Get*", "kinesis:List*", "kinesis:Describe*"],
            "Resource": "*"
        }
    ]
}
```

✅ **Checkpoint:** you can say why `"Resource": "*"` is acceptable for read-only listing but never for writes or deletes in production.

### Step 4 — Create a service role with boto3 (trust + permission policy)

**Users** are for people (long-term credentials). **Roles** are *assumed* temporarily — by services such as Firehose, Redshift, Glue, or an EC2 server — and their credentials rotate automatically. Every role has two policies:

| Policy | Answers | Example |
|---|---|---|
| **Trust policy** | *Who* may assume the role? | `"Principal": {"Service": "ec2.amazonaws.com"}` |
| **Permission policy** | *What* may the role do? | `kinesis:PutRecords` on one stream |

You will build the guided lab's first role, `cloudmetrics-kinesis-producer-role`, with exactly the guided lab's pattern. Save `scripts/lab01_producer_role.py`:

```python
# scripts/lab01_producer_role.py — the guided lab's Role 1, built with the same helper functions
import json

import boto3

from cm_config import REGION, ROLE_PRODUCER, STREAM_NAME, TAGS, account_id

iam = boto3.client("iam")
STREAM_ARN = f"arn:aws:kinesis:{REGION}:{account_id()}:stream/{STREAM_NAME}"


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


role_arn = create_role(ROLE_PRODUCER, trust_service="ec2",
                       description="CloudMetrics API server — write events to Kinesis only")
put_policy(ROLE_PRODUCER, "KinesisProducerPolicy", [{
    "Sid": "KinesisWriteOnly",
    "Effect": "Allow",
    "Action": ["kinesis:PutRecord", "kinesis:PutRecords", "kinesis:DescribeStream"],
    "Resource": STREAM_ARN,
}])
print(role_arn)
```

```bash
python scripts/lab01_producer_role.py
```

**Expected output:**

```
Created cloudmetrics-kinesis-producer-role
arn:aws:iam::123456789012:role/cloudmetrics-kinesis-producer-role
```

Inspect both policies from the CLI:

```bash
aws iam get-role --role-name cloudmetrics-kinesis-producer-role --query Role.AssumeRolePolicyDocument
aws iam get-role-policy --role-name cloudmetrics-kinesis-producer-role --policy-name KinesisProducerPolicy --query PolicyDocument
```

**Expected output:** the first prints the trust policy with `"Service": "ec2.amazonaws.com"` and `"Action": "sts:AssumeRole"`; the second prints the `KinesisWriteOnly` statement.

✅ **Checkpoint:** run `python scripts/lab01_producer_role.py` again — it prints `Reusing cloudmetrics-kinesis-producer-role`. Scripts that can be re-run safely are **idempotent**; every script in SK-04 is written that way.

### Step 5 — Prove the policy with the simulator

Testing a policy by waiting for real `AccessDenied` errors is slow and leaves audit noise. The **policy simulator** evaluates policies without calling the services. Rules: everything is denied by default (*implicit deny*), an `Allow` grants, an explicit `Deny` always wins. Save `scripts/lab01_policy_lab.py`:

```python
# scripts/lab01_policy_lab.py — policy simulator experiments (free, no resources touched)
import json

import boto3

from cm_config import BUCKET, REGION, ROLE_PRODUCER, STREAM_NAME, account_id

iam = boto3.client("iam")
ACCOUNT = account_id()


def decision(**kwargs) -> str:
    return iam.simulate_custom_policy(**kwargs)["EvaluationResults"][0]["EvalDecision"]


print("1) The producer role's real policies")
role_arn = f"arn:aws:iam::{ACCOUNT}:role/{ROLE_PRODUCER}"
stream_arn = f"arn:aws:kinesis:{REGION}:{ACCOUNT}:stream/{STREAM_NAME}"
for action, resource in [("kinesis:PutRecords", stream_arn),
                         ("kinesis:DeleteStream", stream_arn),
                         ("s3:DeleteObject", f"arn:aws:s3:::{BUCKET}/raw/file.parquet")]:
    result = iam.simulate_principal_policy(PolicySourceArn=role_arn, ActionNames=[action],
                                           ResourceArns=[resource])["EvaluationResults"][0]
    print(f"   {action:22} -> {result['EvalDecision']}")

print("2) Bucket-level vs object-level ARNs")
wrong = {"Version": "2012-10-17", "Statement": [
    {"Effect": "Allow", "Action": ["s3:ListBucket", "s3:GetObject"], "Resource": f"arn:aws:s3:::{BUCKET}/*"}]}
right = {"Version": "2012-10-17", "Statement": [
    {"Effect": "Allow", "Action": "s3:ListBucket", "Resource": f"arn:aws:s3:::{BUCKET}"},
    {"Effect": "Allow", "Action": "s3:GetObject", "Resource": f"arn:aws:s3:::{BUCKET}/*"}]}
for name, policy in (("wrong", wrong), ("right", right)):
    for action, resource in [("s3:ListBucket", f"arn:aws:s3:::{BUCKET}"),
                             ("s3:GetObject", f"arn:aws:s3:::{BUCKET}/raw/a.parquet")]:
        result = decision(PolicyInputList=[json.dumps(policy)], ActionNames=[action], ResourceArns=[resource])
        print(f"   {name} policy: {action:14} -> {result}")

print("3) Condition keys: kms:ViaService (the Firehose role in the guided lab uses this)")
kms_policy = {"Version": "2012-10-17", "Statement": [{
    "Effect": "Allow", "Action": "kms:GenerateDataKey", "Resource": "*",
    "Condition": {"StringEquals": {"kms:ViaService": f"s3.{REGION}.amazonaws.com"}}}]}
via_s3 = [{"ContextKeyName": "kms:ViaService", "ContextKeyValues": [f"s3.{REGION}.amazonaws.com"],
           "ContextKeyType": "string"}]
for label, context in (("called through S3", via_s3), ("called directly", [])):
    result = decision(PolicyInputList=[json.dumps(kms_policy)], ActionNames=["kms:GenerateDataKey"],
                      ResourceArns=["*"], ContextEntries=context)
    print(f"   {label:17} -> {result}")
```

```bash
python scripts/lab01_policy_lab.py
```

**Expected output:**

```
1) The producer role's real policies
   kinesis:PutRecords     -> allowed
   kinesis:DeleteStream   -> implicitDeny
   s3:DeleteObject        -> implicitDeny
2) Bucket-level vs object-level ARNs
   wrong policy: s3:ListBucket  -> implicitDeny
   wrong policy: s3:GetObject   -> allowed
   right policy: s3:ListBucket  -> allowed
   right policy: s3:GetObject   -> allowed
3) Condition keys: kms:ViaService (the Firehose role in the guided lab uses this)
   called through S3 -> allowed
   called directly   -> implicitDeny
```

✅ **Checkpoint:** you can explain each line:

- `ListBucket` acts on the **bucket** ARN (`arn:aws:s3:::bucket`). `GetObject`/`PutObject` act on **object** ARNs (`bucket/*`). Mixing them up is the most common IAM bug in data pipelines.
- `kms:ViaService` lets the Firehose role use the key **only when S3 calls KMS on its behalf**, never directly.

### Step 6 — Read the guided lab's IAM model

Open `Labs/SK-04_End_to_End_Guided_Lab.md`, PART 2. Using what you learned, fill in this table in `notes/iam.md` (you will build all four roles in Lab 04):

| Role | Trusted service | Key permissions | One thing it must NOT do |
|---|---|---|---|
| `cloudmetrics-kinesis-producer-role` | ec2 | Put records on one stream | Delete the stream |
| `cloudmetrics-firehose-delivery-role` | firehose | ? | ? |
| `cloudmetrics-glue-etl-role` | glue | ? | ? |
| `cloudmetrics-redshift-spectrum-role` | redshift | ? | ? |

**Expected output:** a completed table (compare with the guided lab's policies).

✅ **Checkpoint:** you can explain why the Redshift role is **read-only** on S3 and why the Firehose role may write only to `raw/` and `rejected/firehose/`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AccessDenied` on `iam:CreateRole` | CLI is not using your admin user | `aws sts get-caller-identity`; re-run `aws configure` |
| `MalformedPolicyDocument` | JSON typo or an invalid ARN | Print `json.dumps(policy, indent=2)` and check brackets and ARN format |
| Simulator says `implicitDeny` where you expected `allowed` | Action or ARN does not match any statement (often bucket vs object ARN) | Compare the resource ARN with the statement's `Resource` character by character |
| A new role is "not authorized" for a few seconds | IAM is eventually consistent | Wait ~10 seconds (the guided lab's script does `time.sleep(10)`) |
| `ModuleNotFoundError: No module named 'cm_config'` | Script run from the wrong folder | Run from `/c/DataEngineering/sk04` as `python scripts/<file>.py` |

## Teardown

IAM is free, but practice resources should not linger. Delete the practice role (the guided lab and Lab 04 recreate it):

```bash
aws iam delete-role-policy --role-name cloudmetrics-kinesis-producer-role --policy-name KinesisProducerPolicy
aws iam delete-role --role-name cloudmetrics-kinesis-producer-role
aws iam get-role --role-name cloudmetrics-kinesis-producer-role
```

**Expected output:** the last command fails with `An error occurred (NoSuchEntity) ... cannot be found` — proof it is gone. (Inline policies must be deleted before the role.)

## Key takeaways

- Services act through **roles**: a trust policy says who may assume it, a permission policy says what it may do.
- Default deny → explicit allow grants → explicit deny always wins.
- Bucket ARN for `ListBucket`; `bucket/*` for object actions.
- Conditions such as `kms:ViaService` narrow *how* a permission may be used.
- Write IAM scripts to be re-runnable (`EntityAlreadyExistsException` → reuse).

## Next

[Lab 02 — S3 Data Lake with KMS Encryption and Lifecycle](Lab_02_S3_Data_Lake_KMS_and_Lifecycle.md)
