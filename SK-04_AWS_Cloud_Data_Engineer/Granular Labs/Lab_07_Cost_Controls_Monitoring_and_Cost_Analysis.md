# Lab 07 — Cost Controls, Monitoring and Cost Analysis

> **Goal:** Put guardrails around the CloudMetrics platform (an SNS alert topic, a tag-filtered budget and CloudWatch alarms), prove the alert path works, build the monthly cost comparison the CTO needs for sign-off, and finish with a scripted clean-slate teardown.
> **Time:** 90–120 min | **Difficulty:** Intermediate | **Prerequisites:** Labs 00–06 (the `scripts/cm_config.py` settings file, the tagged bucket from Lab 02, the roles from Lab 04)
> **Cost:** ~$0.00 — SNS email, the first 10 alarms and the first 2 budgets are free. Nothing in this lab needs the Redshift cluster or the Kinesis stream to be running.
> **Guided lab coverage:** PART 5 (Step 5.1 — `scripts/10_cost_controls.py`, testing the alert path), PART 6 (Step 6.1 — `scripts/11_cost_analysis.py`) and PART 7 (`scripts/99_teardown.py` + verification)

## Learning objectives

By the end of this lab you can:

- Explain the difference between your **account-wide** learner budgets (Lab 00) and a **tag-filtered** budget for one platform.
- Activate a **cost allocation tag** and say why it can take up to 24 hours to become usable.
- Create CloudWatch alarms that notify an SNS topic, and **test** an alarm without waiting for a real incident.
- Build a current-vs-proposed monthly cost table and say which line items grow with data volume and which are roughly fixed.
- Run an idempotent teardown script in dependency order and verify nothing billable is left.

**Terms used in this lab**

| Term | Meaning |
|---|---|
| **SNS topic** | A named "mailbox" in Simple Notification Service. Alarms publish to it; every subscriber (here, your email) receives the message. |
| **Cost allocation tag** | A resource tag (here `CostCenter`) that Billing is told to track, so budgets and Cost Explorer can filter spend by it. |
| **Budget (ACTUAL vs FORECASTED)** | ACTUAL alerts when spend so far crosses a threshold; FORECASTED alerts when AWS predicts the month will cross it. |
| **CloudWatch alarm** | Watches one metric; moves between `OK`, `ALARM` and `INSUFFICIENT_DATA` and sends a notification on change. |
| **Iterator age** | How old the oldest unread Kinesis record is. Rising iterator age = consumers falling behind = stale data. |
| **Idempotent script** | Safe to run again: things already done (or already deleted) are skipped, not errors. |

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Check `ALERT_EMAIL` and what is still running

Open `scripts/cm_config.py` and make sure `ALERT_EMAIL` is an inbox you can open right now — every alert in this lab goes there. Then see what is left from Labs 02–06:

```bash
python -c "import sys; sys.path.insert(0, 'scripts'); from cm_config import ALERT_EMAIL; print(ALERT_EMAIL)"
aws resourcegroupstaggingapi get-resources --region us-east-1 \
    --tag-filters Key=CostCenter,Values=data-platform \
    --query 'ResourceTagMappingList[].ResourceARN'
```

**Expected output:** your email address, then a short list that includes at least `arn:aws:s3:::cloudmetrics-data-lake-<name>` (and the KMS key). The cluster and the stream should **not** be listed — you deleted them at the end of Labs 04 and 06.

✅ **Checkpoint:** `ALERT_EMAIL` is not `you@example.com`, and no Redshift cluster or Kinesis stream is listed. If one is, delete it now using that lab's Teardown section.

### Step 2 — Why two kinds of budget?

Lab 00 created two **account-wide** budgets ($5 and $20): they catch *anything* billing in the account, including resources you forgot to tag. This lab adds a third, **tag-filtered** budget: it only counts resources tagged `CostCenter=data-platform`, which is how a real team answers "what does the data platform cost us?". You need both — the tag-filtered one for accountability, the account-wide ones as the safety net.

Tag filtering only works once `CostCenter` is **activated** as a cost allocation tag, and Billing only offers a tag for activation after it has seen it on a billed resource. That can take up to 24 hours after you first tagged the bucket in Lab 02 — which is why the script below treats a failed activation as "try again tomorrow", not as an error.

✅ **Checkpoint:** you can say in one sentence which budget would catch an *untagged* forgotten EC2 instance (answer: the Lab 00 account-wide ones).

### Step 3 — Create the alert topic, budget and alarms

Save as `scripts/10_cost_controls.py` (identical to guided lab Step 5.1):

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

What each part does:

- **SNS topic + email subscription** — one channel for every alert. `create_topic` returns the existing topic if it is already there, so re-running is safe.
- **Cost allocation tag** — inside `try`: if Billing has not seen `CostCenter` yet you get a message, not a crash.
- **Budget** — $20/month for tagged resources, emailing at 80% ACTUAL and 100% FORECASTED. `DuplicateRecordException` means "already created" — fine on a re-run.
- **Two alarms** — Kinesis iterator age > 15 minutes (data freshness) and Redshift CPU > 80% for 10 minutes (capacity). `put_metric_alarm` overwrites an alarm with the same name, so it is idempotent too.

Run it:

```bash
python scripts/10_cost_controls.py
```

**Expected output:**

```
Topic arn:aws:sns:us-east-1:123456789012:data-platform-alerts — confirm the subscription email in you@yourmail.com
Cost allocation tag CostCenter: Active
Budget cloudmetrics-data-platform created
Budget and alarms configured
```

The tag line may instead read `Could not activate CostCenter yet (...); re-run this script tomorrow` — that is expected on a young account. Now open your inbox and click **Confirm subscription** in the "AWS Notification - Subscription Confirmation" email.

```bash
aws sns list-subscriptions-by-topic \
    --topic-arn "arn:aws:sns:us-east-1:$ACCOUNT_ID:data-platform-alerts" \
    --query 'Subscriptions[].SubscriptionArn'
aws cloudwatch describe-alarms --alarm-names Kinesis-GetRecords-IteratorAge-High Redshift-CPU-High \
    --query 'MetricAlarms[].[AlarmName,StateValue]' --output table
```

**Expected output:** a full subscription ARN (not `PendingConfirmation`), and both alarms in state `INSUFFICIENT_DATA` — correct, because the stream and the cluster they watch do not exist right now. An alarm on a missing resource is not an error; it simply has no data.

✅ **Checkpoint:** subscription confirmed, budget `cloudmetrics-data-platform` visible in **Billing → Budgets**, two alarms listed.

### Step 4 — Test the alert path

An alarm nobody has ever seen fire is a guess, not a control. Force one into `ALARM`:

```bash
aws cloudwatch set-alarm-state --alarm-name Redshift-CPU-High \
    --state-value ALARM --state-reason "Testing the alert path"
```

**Expected output:** no CLI output; within a minute or two an email titled `ALARM: "Redshift-CPU-High" in US East (N. Virginia)` arrives. At the next evaluation (~5 minutes) the alarm goes back to `INSUFFICIENT_DATA` by itself.

✅ **Checkpoint:** you received the ALARM email. Not received? Check spam, then check the subscription is confirmed (Step 3) — an unconfirmed subscription silently drops messages.

### Step 5 — Build the monthly cost comparison

The CTO approves the migration only if the numbers work. Save as `scripts/11_cost_analysis.py` (identical to guided lab Step 6.1). It makes no AWS calls — it is pure arithmetic over production-scale estimates:

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

```bash
python scripts/11_cost_analysis.py
```

**Expected output** (the table first, then):

```
Savings vs current state: $1,476/month ($17,712/year)
Scale ceiling: Current 2TB vs Proposed UNLIMITED
```

`ImportError: Missing optional dependency 'tabulate'` means the venv lacks it: `python -m pip install tabulate`.

**Practice — what scales?** Copy the `proposed` list into a scratch file and mark each line **V** (grows with data volume: S3 storage, S3 requests, Kinesis, Firehose, Spectrum scan, Glue DPU-hours) or **F** (roughly fixed until you resize: Redshift reserved nodes, CloudWatch, data transfer via the VPC endpoint). Then multiply only the V lines by 10 — a first answer to "what if we grow from 5,000 to 50,000 customers?" (guided lab reflection question 4).

✅ **Checkpoint:** your totals are $3,200 (current) and $1,724 (proposed), and you have a V/F mark on every proposed line.

### Step 6 — Write the teardown script

Every SK-04 lab has ended with a manual Teardown. The guided lab uses one script instead, which deletes in **dependency order** (Firehose before the stream it reads, cluster before its subnet group and security group, objects before the bucket, inline policies before roles) and wraps each step so that "already gone" is reported as `[skipped]` instead of stopping the run. Save as `scripts/99_teardown.py` (identical to guided lab Part 7):

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

Read it before you run it and answer for yourself:

1. Why does it *schedule* the KMS key deletion instead of deleting it? (AWS enforces a 7–30 day waiting period; a pending key no longer bills.)
2. Which budgets survive? (Only `cloudmetrics-data-platform` is deleted — your two Lab 00 budgets stay.)
3. Why `bucket.object_versions.delete()` rather than deleting objects? (The bucket is versioned since Lab 02 — every version and delete marker must go before the bucket can.)

✅ **Checkpoint:** you can answer all three questions.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Could not activate CostCenter yet (...)` | Billing has not seen the tag on a billed resource yet | Normal on a new account — re-run `python scripts/10_cost_controls.py` tomorrow |
| `AccessDeniedException` on `budgets` or `ce` calls | IAM access to Billing is switched off for the account (your Lab 00 user is `AdministratorAccess`, so its own policy is not the problem) | Sign in as **root** once: **Account → IAM user and role access to Billing information → Edit → Activate IAM Access**, then sign back in as your IAM user and re-run |
| `InvalidParameterValue` on the subscription | `ALERT_EMAIL` still `you@example.com` or malformed | Fix `scripts/cm_config.py`, re-run the script |
| Test ALARM email never arrives | Subscription still `PendingConfirmation` | Click the confirmation link (check spam), then re-run Step 4 |
| `[CHECK] Redshift cluster: still deleting` in teardown | Cluster deletion takes longer than the waiter allows | Wait 5 minutes and re-run `python scripts/99_teardown.py` |
| `[skipped] S3 bucket: BucketNotEmpty` | New objects arrived between steps (Firehose still running) | Confirm the Firehose stack is gone, re-run the script |

## Teardown

This is the **clean-slate teardown for the whole of SK-04's granular labs**. Run it today unless you go straight on to the guided lab (which recreates everything from scratch anyway — tearing down first is the safer choice):

```bash
python scripts/99_teardown.py
```

**Expected output:** a `[deleted]` line per resource that existed, and `[skipped] … ResourceNotFoundException` / `NoSuchBucket` / `ClusterNotFound` lines for the things you already removed in Labs 03–06 — both are fine. Then verify:

```bash
aws kinesis list-streams --query 'StreamNames'                          # []
aws firehose list-delivery-streams --query 'DeliveryStreamNames'        # []
aws redshift describe-clusters --query 'Clusters[].ClusterIdentifier'   # []
aws s3 ls | grep cloudmetrics || echo "no cloudmetrics buckets"
aws cloudwatch describe-alarms --alarm-names Kinesis-GetRecords-IteratorAge-High Redshift-CPU-High \n    --query 'MetricAlarms[].AlarmName'                                  # []
aws budgets describe-budgets --account-id "$ACCOUNT_ID" --query 'Budgets[].BudgetName'
```

**Expected output:** empty lists, `no cloudmetrics buckets`, and only your two Lab 00 budgets in the last list. The KMS key shows `PendingDeletion` for 7 days and the tagging API may list deleted resources for a few hours — both are fine. Check **Billing → Bills** tomorrow.

## Key takeaways

- Two layers of budget: account-wide as the safety net, tag-filtered for per-platform accountability.
- Every alarm notifies one tested channel — force it into `ALARM` once and see the email.
- A cost case separates volume-driven lines from fixed lines; that split is what answers "what if we grow 10x?".
- Teardown is code too: dependency-ordered, idempotent, and followed by verification.

## Next

[SK-04 End-to-End Guided Lab](../Labs/SK-04_End_to_End_Guided_Lab.md) — build the whole CloudMetrics platform in one sitting, then the [SK-04 capstone project](../Project/SK-04_Capstone_Project.md).
