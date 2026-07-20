# Lab_03 — Terraform Fundamentals

> **Time:** 4–5 hours | **Cost: < $0.10** (same S3 pennies as Lab_02 — Terraform itself is free)
>
> **Cost Check:** Terraform creates the same near-free S3 resources as Lab_02. The real cost lesson here is the opposite direction: `terraform destroy` becomes your one-command teardown for every future lab. Nothing here bills while idle beyond stored bytes.

---

## 1. Environment Setup

Verify:

```powershell
terraform -version          # v1.x (installed in Lab_00)
aws sts get-caller-identity # admin user
aws s3 ls                   # the three swifthaul buckets from Lab_02
```

Create the working folder and open VS Code (with the HashiCorp Terraform extension from Lab_00):

```powershell
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws\terraform
mkdir lake; cd lake
code .
```

**One decision before we start:** Terraform cannot politely adopt hand-made resources without `import`, which is an advanced topic. The clean beginner path is: **delete the Lab_02 buckets, rebuild them as code.** That's not wasted work — proving the lake is reproducible from text *is* the lesson. Run the Lab_02 full-teardown commands now (empty + `rb` all three buckets), then verify `aws s3 ls` shows none.

---

## 2. Business Context

SwiftHaul asks the question every client asks: *"If your laptop dies, or we need a staging copy of this environment, how long to rebuild?"* Clicked-together infrastructure: days of guesswork. Infrastructure as code: `terraform apply`, minutes, identical every time.

In industry, IaC is non-negotiable: environments are reviewed in pull requests, promoted dev→staging→prod from the same code, and audited from Git history. Consumers: your whole team, CI/CD systems, auditors, disaster-recovery plans. If it fails — "snowflake" environments drift apart, staging tests pass while prod breaks, and nobody can say what's actually deployed. Terraform is the market leader; CloudFormation (AWS-only), Pulumi (general-purpose languages), and CDK are the main alternatives.

---

## 3. Concept Explanation

- **Declarative, not imperative:** you describe the *end state* ("a bucket named X with versioning"), and Terraform computes the steps. Contrast with your Lab_02 CLI commands (imperative: exact steps, no memory).
- **HCL (HashiCorp Configuration Language):** the `.tf` file syntax — blocks, arguments, expressions.
- **Provider:** a plugin that knows how to talk to one platform (here `hashicorp/aws`). Downloaded by `terraform init`.
- **Resource:** one managed object (`aws_s3_bucket`). **Data source:** read-only lookup of something Terraform doesn't manage.
- **State (`terraform.tfstate`):** Terraform's record of what it created, mapping code names to real ARNs/IDs. **Never hand-edit it.** If state and reality disagree, `plan` shows drift. Teams store state remotely (S3 + DynamoDB locking) so they don't overwrite each other — we'll discuss, not build, that today.
- **The core loop:** `init` (download providers) → `plan` (preview diff) → `apply` (make it so) → `destroy` (remove everything in state).
- **Variables / outputs:** parameterize inputs (your name, region), export results (bucket ARNs) for humans and other modules.

Pros: reproducible, reviewable, self-documenting, one-command teardown. Cons: state management complexity, a learning curve, and it only manages what it knows about.

---

## 4. Step-by-Step Implementation

### Step 1 — Provider and version pinning

Create `providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      project = "swifthaul"
      env     = "dev"
      managed = "terraform"
    }
  }
}
```

**Why pin versions?** `~> 5.0` means "any 5.x" — a future provider v6 with breaking changes can't silently wreck your plan. **`default_tags`** stamps every resource, which powers per-project cost reporting.

### Step 2 — Variables

Create `variables.tf`:

```hcl
variable "region" {
  description = "AWS region for all resources"
  type        = string
  default     = "eu-west-1"
}

variable "owner" {
  description = "Your name — lowercase, used in globally-unique bucket names"
  type        = string
}

variable "zones" {
  description = "Lake zones to create"
  type        = list(string)
  default     = ["raw", "processed", "curated"]
}
```

And `terraform.tfvars` (values for this deployment):

```hcl
owner = "francis"   # <- your name
```

**Why separate files?** `variables.tf` declares (committed to Git); `terraform.tfvars` supplies environment-specific values (often gitignored when it contains anything sensitive).

### Step 3 — The lake, as code

Create `main.tf`:

```hcl
resource "aws_s3_bucket" "zone" {
  for_each = toset(var.zones)
  bucket   = "swifthaul-${var.owner}-${each.key}"
}

resource "aws_s3_bucket_versioning" "raw" {
  bucket = aws_s3_bucket.zone["raw"].id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "zone" {
  for_each                = aws_s3_bucket.zone
  bucket                  = each.value.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "raw" {
  bucket = aws_s3_bucket.zone["raw"].id
  rule {
    id     = "raw-cost-control"
    status = "Enabled"
    filter {
      prefix = "telemetry/"
    }
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }
}
```

Read it aloud: `for_each` stamps one bucket per zone; versioning and lifecycle apply only to raw — exactly your Lab_02 decisions, now permanent and reviewable.

### Step 4 — Outputs

Create `outputs.tf`:

```hcl
output "bucket_arns" {
  description = "ARNs of all lake buckets"
  value       = { for k, b in aws_s3_bucket.zone : k => b.arn }
}
```

### Step 5 — init

```powershell
terraform init
```

**Expected:** "Terraform has been successfully initialized!" and a new `.terraform\` folder + `.terraform.lock.hcl` (the provider lock file — commit it if using Git).
**Troubleshooting:** proxy/TLS errors on corporate networks → set `HTTPS_PROXY`; "Failed to query available provider packages" → typo in `source`.

### Step 6 — plan (read every line!)

```powershell
terraform plan
```

**Expected:** `Plan: 8 to add, 0 to change, 0 to destroy.` (3 buckets + 3 public-access blocks + versioning + lifecycle). Read the diff — green `+` lines are creations. **This reading habit is the skill.** A production incident is one unread `-/+ destroy and recreate` line away.
**Troubleshooting:** `No value for required variable "owner"` → your `terraform.tfvars` is missing or misnamed.

### Step 7 — apply

```powershell
terraform apply
# type: yes
aws s3 ls   # verify: three swifthaul buckets exist again
```

**Expected:** `Apply complete! Resources: 8 added.` plus your `bucket_arns` output.
**Common mistake:** `BucketAlreadyOwnedByYou` → you didn't fully delete the Lab_02 buckets; empty and remove them, then re-apply.

### Step 8 — Experience drift and idempotency

Run `terraform plan` again. **Expected:** `No changes.` — applying twice is safe: **idempotency**, the property every pipeline in this module chases.

Now cause drift: in the console, suspend versioning on the raw bucket. Then:

```powershell
terraform plan
# Expected: 1 to change — Terraform proposes putting versioning back
terraform apply
```

Terraform just *healed* manual tampering. This is why teams forbid console changes to Terraform-managed resources.

### Step 9 — Re-upload sample data

Terraform manages infrastructure, not data. Re-run the three `aws s3 cp` commands from Lab_02 Step 5 to repopulate `swifthaul-<name>-raw`. **Verify** with `aws s3 ls ... --recursive`.

### Step 10 — Read the state file (don't touch)

```powershell
terraform state list
terraform state show aws_s3_bucket.zone[\"raw\"]
```

**Expected:** all 8 resources listed; the show command prints real attributes (ARN, region). Open `terraform.tfstate` in VS Code, look, close it **without editing**. Note it can contain sensitive values — never commit state to public Git.

**Remote state (concept, not built today):** teams keep state in an S3 bucket with a DynamoDB table for locking, so two engineers can't apply simultaneously. Config lives in a `backend "s3" {}` block. Know this for interviews; local state is fine for a solo learner.

---

## 5. Production Engineering Practices

- **Code review of infrastructure:** `plan` output pasted into a pull request is the industry ritual. *Failure story:* an engineer applied an unreviewed change where a renamed resource meant destroy-and-recreate of a production database. The `-/+` was in the plan; nobody read it.
- **Parameterization:** the same code deploys `swifthaul-francis-*` and `swifthaul-anna-*` from one variable — the mechanism behind dev/staging/prod parity.
- **Modularity:** at ~3 files this is fine flat; real projects split into modules (`modules/lake`, `modules/glue`). Your Project may reuse this folder as a module.
- **Secrets:** none appear in this code — credentials come from `~\.aws\credentials` at runtime. Never write keys into `.tf` files.
- **Teardown as a feature:** `terraform destroy` turns cost-safety from a checklist into a single audited command — the reason this module insisted on IaC.

---

## 6. Reflection

**What you learned:** declarative IaC, HCL, providers, plan/apply lifecycle, variables/outputs, `for_each`, state, drift detection, idempotency.

**Interview questions:**

1. *What is Terraform state and why does it matter?* — The mapping between code and real resource IDs; without it Terraform can't diff or destroy. Hand-editing it desynchronizes code from reality.
2. *How do teams share state safely?* — Remote backend (S3) with locking (DynamoDB) so concurrent applies can't corrupt it.
3. *`plan` vs `apply`?* — Plan computes and displays the diff without changing anything; apply executes it. Plans should be reviewed like code.
4. *What is drift and how do you handle it?* — Reality diverging from state via manual changes; detect with `plan`, fix by re-applying or updating code — and culturally, ban console edits.
5. *Declarative vs imperative IaC?* — Declare end state (Terraform) vs script steps (CLI/bash). Declarative gives idempotency and diffing for free.
6. *Why pin provider versions?* — Prevent breaking provider upgrades from changing behavior between runs; the lock file makes builds reproducible.
7. *Terraform vs CloudFormation?* — Terraform: multi-cloud, huge ecosystem, state you manage. CloudFormation: AWS-native, state managed by AWS, deeper AWS integration but AWS-only.
8. *Is `terraform apply` idempotent?* — Yes: same code + same reality = no changes. That property is why IaC is safe to run repeatedly.

**Trap:** "Terraform deleted my bucket — how?" Expected reasoning: a rename in code = destroy+create; the plan said so. The lesson interviewers want: *always read the plan.*

**Key takeaways:** if it isn't in code, it doesn't exist; read every plan; state is sacred; idempotency is the property everything in data engineering aims for.

---

## Teardown

You now have the one-command version. If ending your session (or the module):

```powershell
# Empty buckets first — Terraform won't destroy non-empty buckets by default:
$name = "francis"
aws s3 rm s3://swifthaul-$name-raw --recursive
# raw is versioned: also purge versions via the Lab_02 delete-objects command, then:
terraform destroy   # type: yes
aws s3 ls           # Expected: no swifthaul buckets
aws resourcegroupstaggingapi get-resources --region eu-west-1   # Expected: empty
```

**If continuing straight to Lab_04:** keep everything applied (cost: pennies) — Lab_04 builds Glue and Athena on top of these exact buckets and adds them to this same Terraform folder.

**Next:** [Lab_04_Glue_ETL_and_Athena.md](Lab_04_Glue_ETL_and_Athena.md).
