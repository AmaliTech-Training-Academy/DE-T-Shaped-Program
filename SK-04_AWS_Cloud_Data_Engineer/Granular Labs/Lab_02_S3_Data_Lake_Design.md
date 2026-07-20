# Lab_02 — S3 Data Lake Design

> **Time:** 3–4 hours | **Cost: < $0.10**
>
> **Cost Check:** S3 charges ~$0.023/GB-month for storage plus fractions of a cent per thousand requests. Our sample data is a few megabytes — effectively free. **Nothing in this lab bills while idle** except stored bytes (pennies). Versioning keeps deleted copies, which is why teardown includes emptying versions.

---

## 1. Environment Setup

Verify prior state:

```powershell
aws sts get-caller-identity                          # admin user
aws iam list-attached-group-policies --group-name swifthaul-data-engineers   # policy from Lab_01
```

Nothing new to install. Create a data folder and three tiny sample files simulating one depot's nightly drop:

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws\sample-data
@'
truck_id,ts,lat,lon,speed_kmh,fuel_pct,engine_temp_c
T1001,2026-07-06T21:14:02Z,33.749,-84.388,87,64,92
T1002,2026-07-06T21:14:05Z,33.755,-84.390,0,71,88
T1003,2026-07-06T21:14:09Z,33.812,-84.401,102,58,95
'@ | Out-File -Encoding utf8 telemetry_atlanta_2026-07-06.csv
```

Make two more copies with different depot names (`dallas`, `denver`) — edit a few values so files differ. **Verify:** `ls` shows three CSVs.

**New terms:** **S3 (Simple Storage Service)** — AWS's object store: you upload files ("objects") into "buckets" and address them by key (path-like name). **Data lake** — a store of raw and refined files organized so many tools can query them.

---

## 2. Business Context

SwiftHaul's 40 depots need somewhere to upload nightly telemetry. Requirements from the client: durable (losing a night's data = losing billable-mile evidence), cheap (10,000 trucks generate terabytes over time), secure (GPS traces are sensitive), and *organized* so that Glue and Athena (Lab_04) can find the right day's data without scanning everything.

In industry, S3 (or its clones — you used **MinIO** in SK-03, which is an S3-compatible server) is the default landing zone for nearly every batch pipeline. Consumers: ETL jobs, analysts via Athena, ML teams, auditors. If it fails — wrong permissions leak customer GPS data (regulatory fines), or an unpartitioned layout makes every query scan the full lake (costs explode, dashboards time out).

---

## 3. Concept Explanation

### Buckets, keys, and the "folders" illusion

S3 has no real folders. `raw/depot=atlanta/year=2026/month=07/day=06/telemetry.csv` is one flat *key*; the console just renders `/` as folders. This matters because "renaming a folder" actually copies every object.

### The three-zone lake (you built this in SK-03 on MinIO)

| Zone | Content | Mutability |
|---|---|---|
| **raw** | Files exactly as delivered — bad rows and all | Immutable, append-only |
| **processed** | Cleaned, deduped, converted to Parquet | Rebuildable from raw |
| **curated** | Business-level aggregates ready for BI | Rebuildable from processed |

Why keep raw forever? Because it's your replay button: when (not if) your ETL has a bug, you re-run from raw. Deleting raw is deleting your undo history.

### Hive-style partitioning

Keys like `year=2026/month=07/day=06/` aren't decoration — query engines (Athena, Spark, Glue) parse `column=value` segments as **partition columns** and skip non-matching prefixes entirely. Query one day → scan one day. Decision framework: partition on what you *filter* by (date, depot), keep cardinality moderate (partitions of at least ~100 MB ideally; thousands of tiny partitions hurt), and put the most-filtered column first.

### Governance features

- **Versioning** — S3 keeps every overwritten/deleted object as a version. Protection against accidental deletes and *duplicate-file* overwrites (project-relevant!). Cost: old versions bill until expired.
- **SSE (Server-Side Encryption)** — SSE-S3 (free, AWS-managed keys, now default) vs SSE-KMS (auditable per-key control, per-request cost). We use SSE-S3.
- **Block Public Access** — four account/bucket switches that make public exposure impossible. Always on for data lakes.
- **Lifecycle rules** — automatic transitions (Standard → Infrequent Access → Glacier) and expirations. This is how lakes stay cheap at petabyte scale.

Alternatives to S3: Azure Data Lake Storage, Google Cloud Storage — same concepts, different names.

---

## 4. Step-by-Step Implementation

Bucket names are **globally unique** across all AWS accounts, hence your name in them.

### Step 1 — Create the three buckets

```powershell
$name = "francis"   # your name, lowercase, no dots
aws s3api create-bucket --bucket swifthaul-$name-raw --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
aws s3api create-bucket --bucket swifthaul-$name-processed --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
aws s3api create-bucket --bucket swifthaul-$name-curated --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
aws s3 ls
```

**Expected:** three buckets listed.
**Why `--create-bucket-configuration`?** Any region except `us-east-1` requires it — a famous AWS quirk.
**Troubleshooting:** `BucketAlreadyExists` → someone on Earth took that name; add a suffix. `IllegalLocationConstraintException` → the flag/region mismatch above.

### Step 2 — Verify Block Public Access

New buckets have it on by default; trust but verify:

```powershell
aws s3api get-public-access-block --bucket swifthaul-$name-raw
```

**Expected:** all four flags `true`. If not:

```powershell
aws s3api put-public-access-block --bucket swifthaul-$name-raw --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Step 3 — Enable versioning on raw

```powershell
aws s3api put-bucket-versioning --bucket swifthaul-$name-raw --versioning-configuration Status=Enabled
aws s3api get-bucket-versioning --bucket swifthaul-$name-raw    # Expected: "Status": "Enabled"
```

**Why raw only?** Raw is the irreplaceable zone; processed/curated are rebuildable, and versioning them doubles cost for no recovery value.

### Step 4 — Confirm encryption

```powershell
aws s3api get-bucket-encryption --bucket swifthaul-$name-raw
```

**Expected:** `"SSEAlgorithm": "AES256"` — SSE-S3, on by default since 2023. Note in your decision log: "SSE-S3 chosen over KMS: no per-request cost, no key admin, adequate for training data."

### Step 5 — Upload with partitioned keys

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws\sample-data
aws s3 cp telemetry_atlanta_2026-07-06.csv s3://swifthaul-$name-raw/telemetry/depot=atlanta/year=2026/month=07/day=06/
aws s3 cp telemetry_dallas_2026-07-06.csv  s3://swifthaul-$name-raw/telemetry/depot=dallas/year=2026/month=07/day=06/
aws s3 cp telemetry_denver_2026-07-06.csv  s3://swifthaul-$name-raw/telemetry/depot=denver/year=2026/month=07/day=06/
aws s3 ls s3://swifthaul-$name-raw/telemetry/ --recursive
```

**Expected:** three keys with full `depot=/year=/month=/day=` paths.
**Why this exact shape?** In Lab_04 a Glue crawler will read `depot`, `year`, `month`, `day` as queryable columns straight from these key names.
**Common mistake:** `Year=2026` (capital) or `2026-07` in one segment — partition names are case-sensitive and one `key=value` per segment.

### Step 6 — Watch versioning defeat a duplicate upload

Re-upload the *same* Atlanta file (simulating a depot re-sending):

```powershell
aws s3 cp telemetry_atlanta_2026-07-06.csv s3://swifthaul-$name-raw/telemetry/depot=atlanta/year=2026/month=07/day=06/
aws s3api list-object-versions --bucket swifthaul-$name-raw --prefix telemetry/depot=atlanta --query "Versions[].{Key:Key,IsLatest:IsLatest,VersionId:VersionId}"
```

**Expected:** two versions of the same key, one `IsLatest: true`. Nothing was lost — this is your duplicate-arrival insurance for the Project.

### Step 7 — Add a lifecycle rule

Create `lifecycle.json` in `sample-data\`:

```json
{
  "Rules": [
    {
      "ID": "raw-cost-control",
      "Status": "Enabled",
      "Filter": { "Prefix": "telemetry/" },
      "Transitions": [{ "Days": 90, "StorageClass": "STANDARD_IA" }],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

Apply and verify:

```powershell
aws s3api put-bucket-lifecycle-configuration --bucket swifthaul-$name-raw --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket swifthaul-$name-raw
```

**Why:** after 90 days telemetry moves to Infrequent Access (~45% cheaper); old duplicate versions purge after 30 days. This rule is the difference between a lake that costs $50/month and one that costs $500/month at scale.
**Troubleshooting:** `MalformedXML` → JSON syntax error, usually a missing bracket.

### Step 8 — Test your Lab_01 policy against the real bucket

Console → IAM → Policy Simulator → user `de-<you>` → `s3:GetObject` on `arn:aws:s3:::swifthaul-<name>-raw/telemetry/depot=atlanta/year=2026/month=07/day=06/telemetry_atlanta_2026-07-06.csv` → **allowed**. Then `s3:PutBucketPolicy` → **denied**. Your least-privilege design from Lab_01 holds against real resources.

---

## 5. Production Engineering Practices

- **Immutability of raw:** never let ETL write back into raw. *Failure story:* a media company's job "cleaned raw in place"; a bug nulled a join key and, with no original left, six months of ad data was unrecoverable.
- **Idempotency groundwork:** deterministic keys (`depot=X/year=/month=/day=/filename`) mean a re-run overwrites the same key instead of creating `file (1).csv` chaos — and versioning keeps the history anyway.
- **Validation at the edge:** production lakes pair uploads with checksums (S3 ETag/`--checksum-algorithm SHA256`) so corrupt transfers are detected on arrival — you'll exploit this in the Project's corrupt-file handling.
- **Naming and tagging:** tag buckets (`project=swifthaul`, `env=dev`) — Cost Explorer can then split spend per project. `aws s3api put-bucket-tagging` does it in one line; Terraform will do it automatically in Lab_03.
- **Security defaults:** Block Public Access + SSE + least-privilege policies = the three-lock standard. Every S3 data-breach headline you've read skipped at least one.

---

## 6. Reflection

**What you learned:** buckets/keys vs folders, three-zone lake design, Hive partitioning, versioning as duplicate/delete insurance, SSE, Block Public Access, lifecycle economics.

**Interview questions:**

1. *Why raw/processed/curated zones?* — Raw = immutable replay source; processed = clean standardized format; curated = business-ready. Enables reprocessing and clear ownership.
2. *What is Hive-style partitioning and why use it?* — `col=value/` key segments that engines use for partition pruning, so queries scan only relevant data — the single biggest Athena cost lever.
3. *How would you choose partition columns?* — Columns used in WHERE filters, moderate cardinality, biggest filter first; avoid tiny partitions.
4. *Versioning: benefits and costs?* — Recovers overwrites/deletes; old versions bill until lifecycle-expired — always pair versioning with `NoncurrentVersionExpiration`.
5. *SSE-S3 vs SSE-KMS?* — S3-managed: free, zero admin. KMS: key-level audit + access control, per-request cost and quota. Regulated data usually forces KMS.
6. *Is S3 a file system?* — No: flat object store, keys not folders, no append, no rename (rename = copy+delete).
7. *How do you keep a petabyte lake cheap?* — Lifecycle transitions to IA/Glacier, expire noncurrent versions, columnar formats, compaction of small files.
8. *How does this compare to MinIO from SK-03?* — Same API and concepts; MinIO is self-hosted S3-compatible storage — great dev stand-in, but no 11-nines durability or lifecycle economics.

**Trap:** "Can you make a bucket both public and encrypted for a partner?" Public buckets for data sharing is the wrong answer regardless of encryption — presigned URLs or cross-account roles are.

**Key takeaways:** partition for the queries you'll run; raw is sacred; versioning + lifecycle = safe *and* cheap; verify every security default rather than trusting it.

---

## Teardown

Storage bills (pennies) until deleted. We keep the **buckets** for Lab_03/04 but you may empty large test data. If ending the module early, full teardown is:

```powershell
# Empty including versions (required before deleting a versioned bucket):
aws s3api delete-objects --bucket swifthaul-$name-raw --delete "$(aws s3api list-object-versions --bucket swifthaul-$name-raw --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3 rb s3://swifthaul-$name-raw
aws s3 rb s3://swifthaul-$name-processed --force
aws s3 rb s3://swifthaul-$name-curated --force
```

For now, **keep everything** and just verify what exists and its size:

```powershell
aws s3 ls s3://swifthaul-$name-raw --recursive --summarize
# Total Size should be a few KB — that is < $0.000001/month. Acceptable.
```

**Next:** [Lab_03_Terraform_Fundamentals.md](Lab_03_Terraform_Fundamentals.md) — you'll delete these hand-made buckets and rebuild them as code.
