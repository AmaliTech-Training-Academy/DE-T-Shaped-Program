# Lab_00 — Environment Setup

> **Time:** 3–4 hours | **Cost: $0.00** — everything in this lab (account creation, IAM, Budgets, CLI, Terraform) is free.
>
> **Cost Check:** Nothing in this lab bills. But this lab is where you build the safety net (budget alerts) that protects you in every later lab. Do not skip it.

---

## 1. Environment Setup

This lab *is* the environment setup for the whole module. By the end you will have:

| Item | Version | Purpose |
|---|---|---|
| AWS account | n/a | Where everything runs |
| Root user secured with MFA | n/a | Account safety |
| AWS Budgets alerts at $5 and $20 | n/a | Wallet safety |
| IAM admin user (with MFA) | n/a | Daily-use identity — never root |
| AWS CLI | v2 (latest stable) | Command-line control of AWS |
| Terraform | latest stable (1.x) | Infrastructure as code (Lab_03+) |
| VS Code + extensions | latest | Editing Terraform/PySpark/SQL |
| Project folder | n/a | `C:\Users\<you>\Projects\swifthaul-aws\` |

**Terminology, up front:**

- **AWS (Amazon Web Services)** — a cloud provider: computers, storage, and services you rent by the second instead of buying.
- **Root user** — the email/password you sign up with. It can do *anything*, including delete the account. Treat it like the master key to a building: lock it away, use it almost never.
- **IAM (Identity and Access Management)** — AWS's free system for creating additional users with limited permissions. You'll do all daily work as an IAM user.
- **MFA (Multi-Factor Authentication)** — a second login factor (an app on your phone generating 6-digit codes) so a stolen password alone can't get in.
- **CLI (Command-Line Interface)** — a program that lets you control AWS from PowerShell instead of clicking in the browser.
- **Terraform** — a tool that creates cloud resources from text files ("infrastructure as code"). You'll meet it properly in Lab_03; today you just install it.

macOS/Linux users: everywhere this lab says PowerShell, use your terminal; installs use `brew install awscli terraform` or your distro's package manager. Everything else is identical.

---

## 2. Business Context

Why does a *data engineer* care about account setup? Because in industry:

- **Cloud spend is an engineering responsibility.** Teams have been billed thousands overnight from a forgotten cluster or a leaked access key mined for crypto. The #1 cause of leaked keys is people working as root or committing keys to Git.
- **Security reviews block deployments.** A pipeline built under sloppy IAM won't pass review at any serious company. Recruiters ask IAM questions in *data* engineering interviews precisely because so many candidates can't answer them.
- **Who consumes this work?** Future-you (every lab), your security team (audit trails), and finance (budgets). If it fails: at best a scary bill, at worst a compromised account.

---

## 3. Concept Explanation

**Why not just use root for everything?** Root can't be permission-limited, its actions are harder to attribute, and if its credentials leak the whole account is gone. The industry pattern is: root creates one admin IAM user, gets MFA, and goes in a drawer.

**Why budgets before anything else?** AWS bills in arrears — you find out what you spent *after* you spent it. AWS Budgets emails you when forecast/actual spend crosses a threshold. It's the smoke alarm you install before lighting the stove. Alternatives: Cost Explorer (reactive, not alerting), third-party FinOps tools (overkill here).

**Why CLI *and* console?** The console (website) is great for learning and inspection; the CLI is scriptable, repeatable, and what real automation uses. You'll use both. Terraform then supersedes hand-run CLI for anything permanent.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the AWS account

1. Go to <https://aws.amazon.com/free> → **Create a Free Account**.
2. Use an email you control. Account name: `swifthaul-learning` (or similar).
3. Enter card details (required; you will not be charged if you follow teardown discipline), verify by phone, choose the **Basic (free)** support plan.

**Expected result:** you can sign in at <https://console.aws.amazon.com> as root.
**Common mistake:** picking a "Professional" account type — choose *Personal*.
**Troubleshooting:** card verification can place a temporary $1 hold; it disappears. Activation can take up to 24h (usually minutes).

### Step 2 — Lock down root: MFA, no access keys

1. Sign in as root → click your account name (top right) → **Security credentials**.
2. Under **Multi-factor authentication (MFA)** → **Assign MFA device** → *Authenticator app* → scan the QR with Google Authenticator / Microsoft Authenticator → enter two consecutive codes.
3. Under **Access keys**, confirm there are **none**. Never create root access keys.

**Verify:** sign out, sign back in as root — it must ask for an MFA code.
**Common mistake:** entering the same code twice instead of two *consecutive* codes.

### Step 3 — Budget alerts FIRST ($5 and $20)

1. Console search bar → **Budgets** (under Billing and Cost Management) → **Create budget**.
2. Choose **Customize** → Budget type: **Cost budget**. Name: `guardrail-5`, Period: Monthly, Amount: **$5**.
3. Add an alert threshold at **80% of actual** and **100% of forecasted**, notification email = your email.
4. Repeat for a second budget `guardrail-20` at **$20**.
5. Also enable: Billing preferences → **Receive Free Tier usage alerts**.

**Verify:** both budgets show in the Budgets list; you receive the SNS/email confirmation if prompted (click the confirm link!).
**Why two budgets?** $5 = "investigate", $20 = "something is actively wrong, tear everything down now."
**Common mistake:** creating the budget but never confirming the notification email — you'd get no alerts.

### Step 4 — Create the IAM admin user

1. Console → **IAM** → **Users** → **Create user**. Name: `admin-<yourname>`.
2. Check **Provide user access to the AWS Management Console** → *I want to create an IAM user* → set a strong password, uncheck "must reset".
3. Permissions: **Attach policies directly** → check **AdministratorAccess**. (Yes, admin for now — Lab_01 teaches least privilege and you'll create a scoped data-engineer identity there.)
4. Create, then note the **console sign-in URL** (contains your 12-digit account ID) — bookmark it.
5. Open the new user → **Security credentials** → **Assign MFA device** (same authenticator app, new entry).

**Verify:** sign out of root, sign in at the bookmarked URL as `admin-<yourname>` with MFA. From here on, **never sign in as root again** in this module.
**Common mistake:** losing the account ID — it's on the root account's top-right menu.

### Step 5 — Create access keys for the CLI

1. As the IAM admin user: IAM → Users → your user → **Security credentials** → **Create access key** → use case: *Command Line Interface* → acknowledge → create.
2. You get an **Access key ID** (like `AKIA…`) and a **Secret access key** (shown once). Keep this tab open for Step 7.

**Never** paste these into any file that could reach Git, chat, or a screenshot.

### Step 6 — Install AWS CLI v2 on Windows

In PowerShell (as your normal user):

```powershell
winget install --id Amazon.AWSCLI -e
```

Close and reopen PowerShell, then verify:

```powershell
aws --version
# Expected: aws-cli/2.x.x Python/3.x Windows/10 exe/AMD64
```

**No winget?** Download the MSI from <https://awscli.amazonaws.com/AWSCLIV2.msi> and run it.
**Troubleshooting:** `aws : The term 'aws' is not recognized` → you didn't reopen PowerShell, or PATH wasn't updated — log off/on, or check `C:\Program Files\Amazon\AWSCLIV2\` exists.

### Step 7 — Configure the CLI

```powershell
aws configure
# AWS Access Key ID:     <paste from Step 5>
# AWS Secret Access Key: <paste from Step 5>
# Default region name:   eu-west-1
# Default output format: json
```

This writes `C:\Users\<you>\.aws\credentials` and `.aws\config`. **Verify** — the single most useful AWS command:

```powershell
aws sts get-caller-identity
```

Expected output (your numbers/name will differ):

```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/admin-francis"
}
```

**Common mistakes:** swapping key ID and secret; region typo (`eu-west1`). **Troubleshooting:** `InvalidClientTokenId` → key pasted wrong or deactivated; re-create the key.

### Step 8 — Install Terraform

```powershell
winget install --id Hashicorp.Terraform -e
```

Reopen PowerShell:

```powershell
terraform -version
# Expected: Terraform v1.x.x on windows_amd64
```

If winget offers nothing, download the Windows AMD64 zip from <https://developer.hashicorp.com/terraform/install>, extract `terraform.exe` to `C:\terraform\`, and add that folder to PATH (Settings → System → About → Advanced system settings → Environment Variables).

### Step 9 — VS Code + project folder

```powershell
winget install --id Microsoft.VisualStudioCode -e
mkdir C:\Users\$env:USERNAME\Projects\swifthaul-aws
cd C:\Users\$env:USERNAME\Projects\swifthaul-aws
mkdir terraform, glue-scripts, sample-data, notes
code .
```

In VS Code install extensions: **HashiCorp Terraform**, **AWS Toolkit** (optional), **Python**.

**Verify:** folder tree exists; `code .` opens VS Code in the project.

### Step 10 — Dry-run sanity check (free)

```powershell
aws s3 ls          # lists buckets — empty output is correct (you have none yet)
aws iam list-users # should show your admin user
```

Both returning without error proves credentials, region, and CLI all work.

---

## 5. Production Engineering Practices

- **Secrets management:** access keys live only in `~\.aws\credentials`, never in code. *Failure story:* a student committed keys to a public GitHub repo; bots found them in under 4 minutes and launched GPU instances for crypto mining — $12,000 in a weekend. AWS often forgives first offenses, but don't test that.
- **Least privilege (preview):** we granted AdministratorAccess for bootstrap convenience; Lab_01 replaces daily work with a scoped policy. In production, humans rarely hold standing admin — they *assume roles* temporarily.
- **Config management:** the `.aws\config` file supports named profiles (`aws configure --profile work`) — how professionals juggle multiple accounts.
- **Monitoring from day zero:** budgets are monitoring. The pattern "alert before act" repeats in Lab_06 with CloudWatch alarms.
- **Documentation:** start `notes\decisions.md` in your project folder. Write one line: "2026-07-07: region eu-west-1 chosen; budgets $5/$20." Real teams keep decision logs; graders love them.

---

## 6. Reflection

**What you learned:** account hygiene, root vs IAM, MFA, budget alarms, CLI and Terraform installation, identity verification with STS.

**Interview questions:**

1. *Why should the root user never be used for daily work?* — It can't be permission-scoped, its compromise is unrecoverable in scope, and best practice is MFA + no access keys + break-glass only.
2. *What does `aws sts get-caller-identity` do?* — Returns the account, user ID, and ARN of the credentials in use; the standard "who am I" check.
3. *How would you prevent surprise cloud bills?* — Budgets with actual+forecast alerts, free-tier alerts, tagging, and teardown discipline; Cost Explorer for investigation.
4. *Where should AWS access keys be stored?* — In the credentials file or a secrets manager/SSO session — never in code or version control. Prefer IAM Identity Center (SSO) or roles over long-lived keys.
5. *What is MFA and why does it matter?* — A second authentication factor; it neutralizes password theft, the most common compromise vector.
6. *What's the difference between the console, the CLI, and Terraform?* — Manual UI vs scriptable commands vs declarative, versioned infrastructure definitions.
7. *What is an ARN?* — Amazon Resource Name: the globally unique identifier of any AWS resource (`arn:aws:iam::123456789012:user/x`). Dissected fully in Lab_01.

**Common interview trap:** "Have you ever used root access keys for automation?" The only right answer is no, with the reason.

**Key takeaways:** budgets before builds; root goes in a drawer; verify identity with STS; one project folder, one region, forever (well, for this module).

---

## Teardown

Nothing to tear down — nothing bills. But end every session with the habit anyway:

```powershell
aws resourcegroupstaggingapi get-resources --region eu-west-1
# Expected now: an empty ResourceTagMappingList
```

If that list is empty and Budgets shows $0.00 forecast, you're clean. **Next:** [Lab_01_AWS_Foundations_and_IAM.md](Lab_01_AWS_Foundations_and_IAM.md).
