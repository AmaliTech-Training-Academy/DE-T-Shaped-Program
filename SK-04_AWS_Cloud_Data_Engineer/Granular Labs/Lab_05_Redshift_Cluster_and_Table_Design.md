# Lab 05 — Redshift Cluster, Networking and Table Design

> **Goal:** Launch the guided lab's private, encrypted Redshift cluster with a managed password, run SQL through the Redshift Data API, and create the CloudMetrics star schema with deliberate distribution, sort and compression choices.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 04 (the KMS key and `cloudmetrics-redshift-spectrum-role` must exist)
> **Cost:** ~$3–4 — a 1-node `ra3.xlplus` bills **~$1.09 per hour while running** (check the AWS Pricing Calculator). **Pause or delete the cluster in Teardown.** Managed password secret: $0.40/month prorated.
> **Guided lab coverage:** PART 4 — Step 4.1 (subnet group, security group, managed password, cluster) and Step 4.2 (`rs.py` Data API helper, DDL with DISTKEY/SORTKEY/ENCODE, staging schema, `dim_date`)

## Learning objectives

By the end of this lab you can:

- Explain provisioned Redshift: leader node, compute nodes, slices, RA3 managed storage — and its cost model (pause, resume, delete).
- Place a cluster in a VPC with a subnet group and a security group, keep it private, and keep its password out of your code with Secrets Manager.
- Run SQL from Python with the **Redshift Data API** (`execute_statement`, `describe_statement`, `get_statement_result`, `batch_execute_statement`).
- Choose DISTSTYLE (KEY / ALL / EVEN / AUTO), sort keys and column encodings (AZ64, ZSTD, BYTEDICT, RAW) and justify them.

**Start every session** (Git Bash):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

## Steps

### Step 1 — Redshift concepts and cost model

| Concept | Meaning |
|---|---|
| **Cluster** | A leader node (plans queries) + compute nodes (store data and execute) |
| **Slice** | A share of a compute node's CPU/memory/disk; `ra3.xlplus` has 2 slices per node |
| **RA3 managed storage** | Data lives in S3-backed storage; nodes cache hot data on local SSD — compute and storage scale separately |
| **Provisioned vs Serverless** | Provisioned = you choose nodes and pay per node-hour (the guided lab); Serverless = pay per query compute |
| **Pause / resume** | A paused cluster stops compute billing; storage still bills |

The guided lab designs CloudMetrics production for **3 nodes (6 slices)** but builds **1 node** to learn on — the SQL is identical; the cost is one third.

**Expected output:** none — this step is reading.

✅ **Checkpoint:** you can say what happens to the bill when the cluster is (a) running and idle, (b) paused, (c) deleted.

### Step 2 — Networking: VPC, subnets, security group

A **VPC** is your private network in AWS. Every account has a **default VPC** with one **subnet** per AZ. Redshift needs a **cluster subnet group** (which subnets it may use) and a **security group** (a firewall). We keep the cluster **private**: `PubliclyAccessible=False` and a security group with **no inbound rules**. You will still query it — through the **Data API**, an HTTPS AWS API authorised by IAM, which needs no open port.

```bash
aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[].VpcId'
aws ec2 describe-subnets --filters Name=default-for-az,Values=true --query 'length(Subnets)'
```

**Expected output:**

```
[
    "vpc-0a1b2c3d4e5f67890"
]
6
```

✅ **Checkpoint:** a default VPC with 6 default subnets exists. (If the first list is empty, see Troubleshooting.)

### Step 3 — Create the cluster: `scripts/07_create_cluster.py`

Read the script before running it:

- Subnet group + security group are created only if missing (idempotent).
- `ManageMasterPassword=True` — Redshift generates the admin password and stores it in **Secrets Manager**. No password in code, chat or Git.
- `IamRoles` + `DefaultIamRoleArn` attach `cloudmetrics-redshift-spectrum-role`, so SQL can say `IAM_ROLE default` for COPY and Spectrum.
- `Encrypted=True, KmsKeyId=…` — the warehouse is encrypted with the same CMK as the lake.
- The waiter blocks until the cluster is `available` (10–15 minutes).

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

```bash
python scripts/07_create_cluster.py
aws redshift describe-clusters --cluster-identifier "$CLUSTER_ID" \
    --query 'Clusters[0].{Status:ClusterStatus,Node:NodeType,Nodes:NumberOfNodes,Public:PubliclyAccessible,Encrypted:Encrypted,Secret:MasterPasswordSecretArn}'
```

**Expected output:**

```
Cluster creation initiated: creating
Waiting for the cluster to become available (10-15 minutes)...
Cluster cloudmetrics-prod is available — it now bills by the hour. Pause it when you step away.
{
    "Status": "available",
    "Node": "ra3.xlplus",
    "Nodes": 1,
    "Public": false,
    "Encrypted": true,
    "Secret": "arn:aws:secretsmanager:us-east-1:123456789012:secret:redshift!cloudmetrics-prod-awsuser-AbCdEf"
}
```

✅ **Checkpoint:** status `available`, not public, encrypted, and a Secrets Manager ARN instead of a password. **The cluster now bills ~$1.09/hour.**

### Step 4 — The Data API helper: `scripts/rs.py`

The **Redshift Data API** runs SQL asynchronously: `execute_statement` returns an **Id** immediately; you poll `describe_statement` until `Status` is `FINISHED` (or `FAILED`/`ABORTED`, with an `Error` message); if `HasResultSet` is true, `get_statement_result` returns `ColumnMetadata` and `Records` (each cell is a dict like `{"stringValue": "x"}`, `{"longValue": 3}` or `{"isNull": true}`). `batch_execute_statement` runs several statements as **one transaction**. `DbUser` gives you temporary database credentials through IAM — no password needed.

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

```bash
python -c "import sys; sys.path.insert(0, 'scripts'); from rs import run, show; show(run('SELECT current_user, current_database()'))"
```

**Expected output:**

```
current_user | current_database
awsuser | analytics
```

✅ **Checkpoint:** you ran SQL on a private cluster with no driver, no password and no open port.

### Step 5 — Design and create the schema: `sql/01_schema.sql`

How Redshift lays data out:

| Choice | Options | Rule of thumb |
|---|---|---|
| **DISTSTYLE** | `KEY` (rows with the same key value on the same slice), `ALL` (full copy on every node), `EVEN` (round-robin), `AUTO` | Big fact table: `KEY` on the most-joined/grouped column with many distinct values. Small dimensions: `ALL` so joins never move data |
| **SORTKEY** | `COMPOUND` (column order matters), `INTERLEAVED` | The column most queries filter on by range — usually the timestamp. Zone maps then skip blocks |
| **ENCODE** | `AZ64` (numbers, dates, timestamps), `ZSTD` (varied strings), `BYTEDICT` (few distinct values, e.g. `method`), `RAW` (none — e.g. tiny columns, sort keys you scan fully) | Smaller columns = less I/O = faster and cheaper |

The schema also has a **`staging`** table with exactly the Parquet file's columns (COPY lands there first, Lab 06), and a `dim_date` table filled by SQL. Save the guided lab's DDL as `sql/01_schema.sql`:

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

```bash
python scripts/rs.py sql/01_schema.sql
```

**Expected output** (one `->` line per statement, ending with):

```
-> CREATE SCHEMA IF NOT EXISTS analytics
-> CREATE SCHEMA IF NOT EXISTS staging
-> CREATE TABLE IF NOT EXISTS analytics.fact_api_events (
-> CREATE TABLE IF NOT EXISTS staging.api_events_raw (
-> CREATE TABLE IF NOT EXISTS analytics.dim_customer (
-> CREATE TABLE IF NOT EXISTS analytics.dim_date (
-> DELETE FROM analytics.dim_date
-> INSERT INTO analytics.dim_date
-> SELECT COUNT(*) AS dim_date_rows FROM analytics.dim_date
dim_date_rows
4018
```

✅ **Checkpoint:** 4,018 dates (2020-01-01 to 2030-12-31). Run the file a second time — still 4,018: the script is idempotent (`IF NOT EXISTS`, delete-then-insert).

### Step 6 — Inspect what Redshift stored

Save `sql/lab05_inspect.sql`:

```sql
-- Lab 05: inspect the physical design Redshift recorded
SHOW TABLE analytics.fact_api_events;

-- svv_table_info lists only tables that already contain rows (here: dim_date)
SELECT "schema", "table", diststyle, sortkey1, tbl_rows
FROM svv_table_info
WHERE "schema" IN ('analytics', 'staging')
ORDER BY 1, 2;
```

```bash
python scripts/rs.py sql/lab05_inspect.sql
```

**Expected output** (abridged):

```
-> SHOW TABLE analytics.fact_api_events
Show Table DDL statement
CREATE TABLE analytics.fact_api_events ( event_id character varying(36) NOT NULL ENCODE zstd ... ) DISTSTYLE KEY SORTKEY ( event_timestamp );
-> SELECT "schema", "table", diststyle, sortkey1, tbl_rows
schema | table | diststyle | sortkey1 | tbl_rows
analytics | dim_date | ALL | AUTO(SORTKEY) | 4018
```

✅ **Checkpoint:** you can point at `DISTKEY`, `SORTKEY` and each `ENCODE` in the stored DDL and justify it — for example, why `method` is `BYTEDICT` (fewer than 20 distinct values) and `customer_id` is `ZSTD`.

### Step 7 — Pause and resume

```bash
aws redshift pause-cluster --cluster-identifier "$CLUSTER_ID" --query 'Cluster.ClusterStatus'
aws redshift describe-clusters --cluster-identifier "$CLUSTER_ID" --query 'Clusters[0].ClusterStatus'
```

**Expected output:** `"pausing"`, and a few minutes later the second command prints `"paused"`. To continue later:

```bash
aws redshift resume-cluster --cluster-identifier "$CLUSTER_ID" --query 'Cluster.ClusterStatus'
aws redshift wait cluster-available --cluster-identifier "$CLUSTER_ID"
```

✅ **Checkpoint:** you have paused and resumed the cluster once. Make it a habit: **every time you step away, pause.**

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `describe-vpcs` returns `[]` | The account has no default VPC | Console → VPC → **Actions → Create default VPC**, then re-run Step 2 |
| `InvalidParameterCombination` mentioning node count | `NumberOfNodes` passed with `ClusterType="single-node"` | Use the script as written (single-node has no node count) |
| `InsufficientClusterCapacity` | No `ra3.xlplus` capacity in one AZ | Re-run later, or remove that AZ's subnet from the subnet group |
| `Query failed: … permission denied` / `IAM … GetClusterCredentials` | Not using your admin identity | `aws sts get-caller-identity` |
| `rs.py` hangs on `Status` `SUBMITTED` | Cluster paused or still resuming | `aws redshift describe-clusters …` — resume / wait for `available` |
| `ResourceNotFoundException` from the Data API | Wrong `CLUSTER_ID` or region | Check `scripts/cm_config.py` |

## Teardown

**Continuing to Lab 06 today?** Pause the cluster (Step 7) — or leave it running only if you start Lab 06 straight away.

**Not continuing today?** Delete the cluster and its network pieces (Lab 06 recreates them with `python scripts/07_create_cluster.py` + `python scripts/rs.py sql/01_schema.sql`, ~15 minutes):

```bash
aws redshift delete-cluster --cluster-identifier "$CLUSTER_ID" --skip-final-cluster-snapshot --query 'Cluster.ClusterStatus'
aws redshift wait cluster-deleted --cluster-identifier "$CLUSTER_ID"
aws redshift delete-cluster-subnet-group --cluster-subnet-group-name cloudmetrics-subnet-group
SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=cloudmetrics-redshift-sg --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 delete-security-group --group-id "$SG_ID"
aws redshift describe-clusters --query 'Clusters[].ClusterIdentifier'
```

**Expected output:** `"deleting"`, then (after ~5–10 minutes) `[]`. The managed password secret is deleted with the cluster. Keep the bucket, key, Glue table and IAM roles.

## Key takeaways

- A provisioned cluster bills per node-hour whether or not you query it: pause or delete.
- Private by default: subnet group + no-inbound security group + Data API; passwords live in Secrets Manager.
- The Data API is asynchronous: execute → poll → fetch; batches are single transactions.
- DISTKEY for the big join column, ALL for small dimensions, SORTKEY on the range-filtered timestamp, encodings matched to data shape.

## Next

[Lab 06 — Loading Redshift: COPY, Manifests and Spectrum](Lab_06_Redshift_COPY_Manifests_and_Spectrum.md)
