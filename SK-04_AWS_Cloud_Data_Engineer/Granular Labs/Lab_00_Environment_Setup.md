# Lab 00 — Environment Setup

> **Goal:** Set up a safe AWS account, the AWS CLI, Git Bash and a Python virtual environment with boto3, so every later lab and the SK-04 guided lab run without setup surprises.
> **Time:** 120–180 min | **Difficulty:** Beginner | **Prerequisites:** SK-03 completed; a Windows 11 PC with admin rights and a credit/debit card for AWS sign-up
> **Cost:** $0.00 — nothing in this lab bills. You create the budget alerts that protect you in every later lab.
> **Guided lab coverage:** PART 0 (setup, `scripts/cm_config.py`, "every session" commands) and the PREREQUISITES of `Labs/SK-04_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Explain root user vs IAM user, and why MFA and budgets come **before** building anything.
- Use the AWS CLI with your own IAM identity in region `us-east-1` (the region the guided lab uses).
- Run bash commands in **Git Bash** and Python scripts in a **virtual environment**, which is how the guided lab is written.
- Create `scripts/cm_config.py`, the single file holding your name, bucket name and every resource name used in SK-04.

**Terms used in this lab**

| Term | Meaning |
|---|---|
| **AWS account / root user** | The account you sign up with. Root can do *anything*, including closing the account — lock it away. |
| **IAM user** | A named identity with limited permissions. You do all daily work as an IAM user. |
| **MFA** | Multi-factor authentication: a 6-digit code from a phone app, so a stolen password alone cannot get in. |
| **Region** | A geographic group of AWS data centres. SK-04 uses **`us-east-1` (N. Virginia)** everywhere. |
| **AWS CLI** | The `aws` command — controls AWS from a terminal. |
| **boto3** | The AWS SDK for Python — the same API calls as the CLI, from Python code. |
| **Git Bash** | A bash terminal for Windows (installed with Git). The guided lab's shell scripts are bash. |
| **venv** | A Python virtual environment: a project-private set of packages. |

## Steps

### Step 1 — Create the AWS account

1. Go to <https://aws.amazon.com/free> → **Create a Free Account**.
2. Use an email you control. Account name: `cloudmetrics-learning`. Account type: **Personal**.
3. Enter card details (you will not be charged if you follow every Teardown section), verify by phone, choose the **Basic (free)** support plan.

**Expected output:** you can sign in at <https://console.aws.amazon.com> as the root user.

✅ **Checkpoint:** the AWS console home page opens and shows your account name top-right.

### Step 2 — Lock down root: MFA, no access keys

1. Signed in as root → account name (top right) → **Security credentials**.
2. **Multi-factor authentication (MFA)** → **Assign MFA device** → *Authenticator app* → scan the QR code with Google/Microsoft Authenticator → enter **two consecutive** codes.
3. Under **Access keys**, confirm there are **none**. Never create root access keys.

**Expected output:** the MFA section lists one device of type *Virtual*.

✅ **Checkpoint:** sign out and back in as root — you are asked for an MFA code.

### Step 3 — Budget alerts FIRST ($5 and $20)

AWS bills *after* you spend. Budgets email you when spend crosses a threshold — install the smoke alarm before lighting the stove.

1. Console search bar → **Budgets** → **Create budget** → **Customize** → **Cost budget**.
2. Name `guardrail-5`, Period **Monthly**, Amount **$5**. Alerts: **80% of actual** and **100% of forecasted**, email = your email.
3. Repeat for `guardrail-20` at **$20**.
4. **Billing and Cost Management → Billing preferences** → enable **Receive Free Tier usage alerts**.

**Expected output:** the Budgets list shows `guardrail-5` and `guardrail-20`, each with 2 alerts.

✅ **Checkpoint:** both budgets exist. $5 means "investigate"; $20 means "something is running that should not be — run the Teardown now". These two budgets stay for the whole module (Lab 07 adds a third, tag-filtered one).

### Step 4 — Create your IAM admin user

1. Console → **IAM** → **Users** → **Create user**. Name: `admin-<yourname>`.
2. Tick **Provide user access to the AWS Management Console** → *I want to create an IAM user* → set a strong password, untick "must create a new password".
3. **Attach policies directly** → **AdministratorAccess**. (Lab 01 explains why services get narrow roles while this bootstrap human identity is admin.)
4. Create the user. Copy the **console sign-in URL** (it contains your 12-digit account ID) and bookmark it.
5. Open the user → **Security credentials** → **Assign MFA device** (new entry in the same app).

**Expected output:** user `admin-<yourname>` with policy `AdministratorAccess` and one MFA device.

✅ **Checkpoint:** sign out of root, sign in at the bookmarked URL as `admin-<yourname>` with MFA. **Never sign in as root again in this module.**

### Step 5 — Create access keys for the CLI

As `admin-<yourname>`: IAM → Users → your user → **Security credentials** → **Create access key** → *Command Line Interface (CLI)* → acknowledge → **Create**. Keep the tab open; the secret is shown only once.

**Expected output:** an **Access key ID** starting `AKIA` and a **Secret access key**.

✅ **Checkpoint:** you have both values on screen. Never paste them into a file that could reach Git, chat, or a screenshot.

### Step 6 — Install the AWS CLI v2 (PowerShell)

Open **PowerShell** (Start → type PowerShell):

```powershell
winget install --id Amazon.AWSCLI -e
```

Close and reopen PowerShell, then:

```powershell
aws --version
```

**Expected output:**

```
aws-cli/2.x.x Python/3.x.x Windows/10 exe/AMD64
```

✅ **Checkpoint:** `aws --version` prints `aws-cli/2`. (No winget? Install the MSI from <https://awscli.amazonaws.com/AWSCLIV2.msi>.)

### Step 7 — Configure the CLI for us-east-1

```powershell
aws configure
# AWS Access Key ID:     <paste from Step 5>
# AWS Secret Access Key: <paste from Step 5>
# Default region name:   us-east-1
# Default output format: json
aws sts get-caller-identity
```

**Expected output** (your numbers differ):

```json
{
    "UserId": "AIDAEXAMPLEID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-francis"
}
```

✅ **Checkpoint:** the `Arn` ends in `:user/admin-<yourname>` — **not** `:root`. Close the access-key browser tab now.

### Step 8 — Install Git (for Git Bash), Python 3.12 and VS Code

```powershell
winget install --id Git.Git -e
winget install --id Python.Python.3.12 -e
winget install --id Microsoft.VisualStudioCode -e
```

Close and reopen PowerShell, then:

```powershell
git --version
py -3.12 --version
```

**Expected output:**

```
git version 2.x.x.windows.1
Python 3.12.x
```

✅ **Checkpoint:** both versions print. (If you already have Python 3.11+ from SK-01, that works too — use it wherever this module says 3.12.)

### Step 9 — Create the project folder and virtual environment

**Use `C:\DataEngineering\sk04` — not Documents, Desktop or any OneDrive folder.** OneDrive locks files while syncing and causes random "file in use" errors; long OneDrive paths also break some tools.

```powershell
New-Item -ItemType Directory -Force C:\DataEngineering\sk04\scripts, C:\DataEngineering\sk04\sql, C:\DataEngineering\sk04\infra, C:\DataEngineering\sk04\notes
cd C:\DataEngineering\sk04
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install boto3 pandas tabulate
python -c "import boto3, pandas, tabulate; print('boto3', boto3.__version__)"
```

If `Activate.ps1` is blocked ("running scripts is disabled"), run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once (SK-01 Lab 00 explains it) and retry.

**Expected output** (last line; version may be newer):

```
boto3 1.x.x
```

✅ **Checkpoint:** your PowerShell prompt starts with `(.venv)`. `tabulate` is required later by pandas' `to_markdown()` (Lab 07 and guided lab Part 6).

### Step 10 — Meet Git Bash (where the rest of SK-04 runs)

Start menu → **Git Bash**. Then:

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
which python
aws sts get-caller-identity --query Arn --output text
```

**Expected output:**

```
/c/DataEngineering/sk04/.venv/Scripts/python
arn:aws:iam::123456789012:user/admin-francis
```

Differences from PowerShell to remember:

| | PowerShell | Git Bash |
|---|---|---|
| Path to the project | `C:\DataEngineering\sk04` | `/c/DataEngineering/sk04` |
| Activate the venv | `.\.venv\Scripts\Activate.ps1` | `source .venv/Scripts/activate` |
| Continue a long command on the next line | backtick `` ` `` | backslash `\` |
| Paste | Ctrl+V | Shift+Insert (or right-click → Paste) |

✅ **Checkpoint:** `which python` points inside `.venv` and the CLI works from Git Bash.

### Step 11 — Create the shared settings file `scripts/cm_config.py`

Every lab and the guided lab import this file, and bash reads it too, so **your name lives in exactly one place**. Open the folder in VS Code (`code .` in Git Bash) and create `scripts/cm_config.py` — change `NAME` and `ALERT_EMAIL`:

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

Now load it into bash:

```bash
eval "$(python scripts/cm_config.py)"
echo "$BUCKET in $REGION for account $ACCOUNT_ID"
```

**Expected output:**

```
cloudmetrics-data-lake-francis in us-east-1 for account 123456789012
```

✅ **Checkpoint:** the bucket name contains **your** name. Bucket names are global across all AWS customers, so `cloudmetrics-data-lake-francis` can exist only once on Earth.

### Step 12 — The "every session" routine and a cost check

Every time you open Git Bash for SK-04, run these three lines first (the guided lab uses the same routine):

```bash
cd /c/DataEngineering/sk04
source .venv/Scripts/activate
eval "$(python scripts/cm_config.py)"
```

And end every session by checking what exists:

```bash
aws resourcegroupstaggingapi get-resources --region us-east-1 --query 'ResourceTagMappingList[].ResourceARN'
```

**Expected output:**

```
[]
```

✅ **Checkpoint:** an empty list — nothing exists yet. Write one line in `notes/decisions.md`: "Region us-east-1; budgets $5/$20; project at C:\DataEngineering\sk04."

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `aws : The term 'aws' is not recognized` | PowerShell opened before the install finished | Close and reopen PowerShell; check `C:\Program Files\Amazon\AWSCLIV2\` exists |
| `InvalidClientTokenId` / `SignatureDoesNotMatch` | Key ID and secret swapped or mistyped | Re-run `aws configure`; if in doubt create a new key and deactivate the old one |
| `get-caller-identity` shows `:root` | You created root access keys | Delete them (root → Security credentials) and redo Steps 4–7 |
| `Activate.ps1 cannot be loaded` | PowerShell execution policy | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, then retry |
| `python` in Git Bash opens the Microsoft Store or hangs | Windows "App execution alias" stub, or the interactive REPL needs a console | Activate the venv first; for the interactive prompt use `winpty python` |
| `ModuleNotFoundError: No module named 'boto3'` | venv not active | `source .venv/Scripts/activate` (Git Bash) or `.\.venv\Scripts\Activate.ps1` (PowerShell) |
| Budget alert emails never arrive | Notification not confirmed / wrong address | Edit the budget alerts and check the email address |

## Teardown

Nothing bills in this lab — **keep everything** (budgets, admin user, CLI configuration, venv). If you ever suspect your access key leaked: IAM → your user → Security credentials → **Deactivate** the key, then create a new one.

## Key takeaways

- Budgets and MFA come before any resource.
- Daily work happens as an IAM user, never root; verify with `aws sts get-caller-identity`.
- One region (`us-east-1`), one project folder outside OneDrive (`C:\DataEngineering\sk04`), one settings file (`scripts/cm_config.py`).
- Bash blocks run in Git Bash; Python runs inside the activated venv.

## Next

[Lab 01 — AWS Foundations, IAM and boto3](Lab_01_AWS_Foundations_IAM_and_Boto3.md)
