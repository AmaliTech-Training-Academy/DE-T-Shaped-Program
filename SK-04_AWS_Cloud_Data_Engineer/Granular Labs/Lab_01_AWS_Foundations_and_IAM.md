# Lab_01 — AWS Foundations & IAM

> **Time:** 3–4 hours | **Cost: $0.00** — IAM is completely free; we create no billable resources.
>
> **Cost Check:** Zero. IAM users, groups, roles, and policies never bill. Nothing in this lab bills while idle. Still run the teardown verification at the end — habit is the point.

---

## 1. Environment Setup

Verify Lab_00 state before continuing:

```powershell
aws sts get-caller-identity      # returns your admin IAM user ARN
aws --version                    # aws-cli/2.x
terraform -version               # v1.x
```

All three must succeed. Nothing new to install today. You'll work in the console (signed in as `admin-<yourname>`, *not* root) and PowerShell. Create a notes file:

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws
ni notes\lab01-iam.md
```

**Common problem:** `get-caller-identity` shows `:root` in the ARN → you configured root keys. Go back to Lab_00 Step 5 and fix this before proceeding.

---

## 2. Business Context

SwiftHaul is about to give you (a consultant) access to their cloud. Their CTO's first question won't be about Spark — it will be: *"How do we give your team access without giving away the keys to the company?"*

That is IAM. In industry:

- **Every service call is authorized by IAM.** When your Glue job (Lab_04) reads S3, IAM decides yes/no. Most "pipeline broken" tickets in cloud teams are actually IAM `AccessDenied` errors.
- **Auditors demand least privilege.** SOC 2, ISO 27001, GDPR audits all examine who can touch data. A data engineer who can't reason about policies becomes a bottleneck.
- **Consumers of this work:** every teammate, every automated job, every audit. **If it fails:** either people are blocked (overly tight) or a breach exposes customer data (overly loose). The Capital One 2019 breach — 100M records — was fundamentally an over-permissive IAM role.

---

## 3. Concept Explanation

### AWS global infrastructure

- **Region** — a physical cluster of data centers in a geographic area (e.g. `eu-west-1` = Ireland). Regions are isolated from each other; resources mostly live in exactly one region.
- **Availability Zone (AZ)** — one or more discrete data centers within a region with independent power/network. `eu-west-1` has three (`eu-west-1a/b/c`). Multi-AZ = surviving a data-center fire.
- **Edge locations** — CDN endpoints (CloudFront); not relevant to this module.

Why we chose one region and stick to it: cross-region data transfer costs money, and half of beginner confusion is "my bucket disappeared" (wrong region selected in the console's top-right dropdown).

### Shared responsibility model

AWS secures the cloud (buildings, hardware, hypervisor); **you** secure what's *in* the cloud (your data, IAM policies, encryption settings, bucket permissions). One-minute interview version: "AWS guards the building; I lock my own office."

### IAM building blocks

| Term | What it is | Analogy |
|---|---|---|
| **User** | An identity for a person/app with long-term credentials | An employee badge |
| **Group** | A bundle of users sharing permissions | A department |
| **Role** | An identity *assumed temporarily* by a user or AWS service — no permanent credentials | A visitor badge you check out and return |
| **Policy** | A JSON document stating what is allowed/denied | The rules printed on the badge |

Key design fact: **services use roles, humans use users (or better, SSO).** Your Glue job in Lab_04 will run *as a role*.

**Policy evaluation:** everything is implicitly denied; an explicit `Allow` grants; an explicit `Deny` always wins. Alternatives to hand-written JSON: AWS managed policies (fast, coarse) vs customer-managed policies (precise, auditable) — production teams use customer-managed for anything sensitive.

### ARNs

`arn:aws:s3:::swifthaul-francis-raw/depot=atlanta/file.csv`
`arn:aws:iam::123456789012:role/glue-etl-role`

Format: `arn:partition:service:region:account-id:resource`. S3 omits region/account because bucket names are globally unique. Being able to read an ARN out loud is a real interview differentiator.

---

## 4. Step-by-Step Implementation

### Step 1 — Explore regions and AZs (console)

1. Sign in as your IAM admin. Top-right region selector → set **Europe (Ireland) eu-west-1**.
2. Verify AZs from the CLI:

```powershell
aws ec2 describe-availability-zones --region eu-west-1 --query "AvailabilityZones[].ZoneName"
```

**Expected:** `["eu-west-1a", "eu-west-1b", "eu-west-1c"]`.
**Why:** proves your CLI region works and shows you AZ naming.
**Common mistake:** console region drifting back to `us-east-1` — check it every session.

### Step 2 — Read a managed policy before writing one

1. Console → IAM → **Policies** → search `AmazonS3ReadOnlyAccess` → open → **JSON** tab.

You'll see:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:Get*", "s3:List*", "s3:Describe*"],
      "Resource": "*"
    }
  ]
}
```

Dissect it in your notes: `Version` is a fixed schema date (always 2012-10-17, not today's date); `Effect` Allow/Deny; `Action` service:operation with wildcards; `Resource` which ARNs (here: everything — that `*` is exactly what least privilege forbids for writes).

### Step 3 — Write a least-privilege data-engineer policy

We'll scope to buckets prefixed `swifthaul-*` (you create them in Lab_02) plus the read-only actions engineers need for Glue/Athena later.

IAM → Policies → **Create policy** → JSON tab, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LakeBucketAccess",
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::swifthaul-*"
    },
    {
      "Sid": "LakeObjectAccess",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::swifthaul-*/*"
    },
    {
      "Sid": "ReadOnlyVisibility",
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets", "glue:Get*", "athena:Get*", "athena:List*", "athena:StartQueryExecution", "cloudwatch:Get*", "cloudwatch:List*", "logs:Get*", "logs:Describe*", "logs:FilterLogEvents"],
      "Resource": "*"
    }
  ]
}
```

Name it `SwiftHaulDataEngineerPolicy`, description "Scoped lake access for SK-04".

**Why two S3 statements?** Bucket-level actions (`ListBucket`) apply to the bucket ARN; object-level actions (`GetObject`) apply to `bucket/*`. Mixing them up is the single most common IAM bug — you'd get `AccessDenied` on listing even though reads work.
**Verify:** policy saves without the editor flagging errors.
**Troubleshooting:** "Invalid Action" → typo in the service prefix; actions are case-insensitive but must exist.

### Step 4 — Create the group and a data-engineer user

```powershell
aws iam create-group --group-name swifthaul-data-engineers
aws iam attach-group-policy --group-name swifthaul-data-engineers `
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/SwiftHaulDataEngineerPolicy
aws iam create-user --user-name de-$env:USERNAME
aws iam add-user-to-group --group-name swifthaul-data-engineers --user-name de-$env:USERNAME
aws iam list-groups-for-user --user-name de-$env:USERNAME
```

Replace `<ACCOUNT_ID>` with your 12-digit account ID (`aws sts get-caller-identity` shows it).

**Expected:** the last command lists `swifthaul-data-engineers`.
**Why a group, not a direct attach?** The next engineer joins the group in one command; permissions stay uniform and auditable.
**Common mistake:** backtick line-continuation only works in PowerShell; in cmd.exe it fails — use PowerShell.

### Step 5 — Test with the Policy Simulator

1. Console → IAM → Users → `de-<you>` → (right side) **Simulate** (or open <https://policysim.aws.amazon.com>).
2. Select service **S3**, action `GetObject`, resource ARN `arn:aws:s3:::swifthaul-test/file.csv` → Run. **Expected: allowed.**
3. Now simulate `s3:GetObject` on `arn:aws:s3:::someone-elses-bucket/file.csv`. **Expected: denied** — implicit deny, because no statement matches.
4. Simulate `iam:CreateUser`. **Expected: denied.**

**Why:** simulating is how professionals test policies without generating real (and audited) `AccessDenied` events.

### Step 6 — Meet roles (you'll need one in Lab_04)

Create the role Glue will eventually use — free now, ready later:

1. IAM → **Roles** → Create role → Trusted entity: **AWS service** → use case **Glue**.
2. Attach `AWSGlueServiceRole` (managed) for now — you'll tighten it in the Project.
3. Name: `swifthaul-glue-role`. Create.
4. Open it and read the **Trust relationships** tab:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "glue.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

**Concept check:** a role has *two* policy types — the **trust policy** (who may wear the badge) and **permission policies** (what the badge allows). Interviewers love this distinction.

### Step 7 — Record ARNs in your notes

```powershell
aws iam get-role --role-name swifthaul-glue-role --query Role.Arn
aws iam get-user --user-name de-$env:USERNAME --query User.Arn
```

Paste both into `notes\lab01-iam.md`. You will reference the role ARN in Labs 03–04.

---

## 5. Production Engineering Practices

- **Least privilege as iteration:** start narrow, add actions when a real `AccessDenied` appears (CloudTrail tells you exactly which action failed). *Failure story:* a retailer's analytics role had `s3:*` on `*`; a compromised notebook deleted three years of raw data. Scoped writes would have limited it to one bucket.
- **No shared users:** one human = one user. Shared credentials destroy audit trails.
- **Roles over keys for services:** long-lived keys in a Glue job config leak; role credentials rotate automatically every few hours.
- **Naming conventions:** `swifthaul-<purpose>-<thing>` everywhere. Consistent names make policies with `swifthaul-*` wildcards safe and reviews fast.
- **Documentation:** your decision log should now say who has what and why. Auditors ask exactly this.

---

## 6. Reflection

**What you learned:** regions/AZs, shared responsibility, policy JSON anatomy, users vs groups vs roles, trust vs permission policies, the policy simulator, ARNs.

**Interview questions:**

1. *IAM user vs role?* — User: permanent identity with long-term credentials, for humans. Role: temporary, assumed identity with auto-rotating credentials, for services and cross-account access.
2. *Walk me through how a policy is evaluated.* — Default deny → explicit allow grants → explicit deny overrides everything.
3. *Why did your Glue job get AccessDenied on `s3:ListBucket` but not `s3:GetObject`?* — Bucket-level vs object-level ARNs; `ListBucket` must target the bucket ARN, not `bucket/*`.
4. *What is a trust policy?* — The role's statement of who may assume it — separate from what the role can do.
5. *Explain the shared responsibility model.* — AWS: security *of* the cloud (physical, hypervisor). Customer: security *in* the cloud (data, IAM, config).
6. *What's an AZ and why do multi-AZ deployments matter?* — Independent data centers within a region; they isolate power/network failures.
7. *How would you test a policy before deploying?* — IAM Policy Simulator, or a sandbox account, plus CloudTrail review after rollout.
8. *Is `"Resource": "*"` ever acceptable?* — For read-only/list actions sometimes; never for write/delete on data stores in production.

**Trap:** "Just give the role AdministratorAccess to unblock the pipeline — would you?" Correct answer: no; diagnose the missing action via CloudTrail and add only that.

**Key takeaways:** everything is denied until allowed; humans get users, services get roles; bucket vs object ARNs; simulate before you ship.

---

## Teardown

Nothing bills, so nothing must be destroyed. Keep `SwiftHaulDataEngineerPolicy`, the group, `de-<you>`, and `swifthaul-glue-role` — Labs 02–04 use them. Verify cleanliness anyway:

```powershell
aws resourcegroupstaggingapi get-resources --region eu-west-1
# Expected: empty list — IAM is global and free, S3 not created yet
```

**Next:** [Lab_02_S3_Data_Lake_Design.md](Lab_02_S3_Data_Lake_Design.md) — the first billable (pennies) lab.
