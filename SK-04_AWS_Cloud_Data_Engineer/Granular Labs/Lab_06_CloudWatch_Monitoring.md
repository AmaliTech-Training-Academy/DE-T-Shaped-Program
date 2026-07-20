# Lab 06 — CloudWatch Monitoring: Logs, Metrics, Alarms, and Dashboards as Code

> **Module:** SK-04 AWS Cloud Data Engineer
> **Estimated time:** 3–4 hours
> **Cost estimate:** < $1. First 10 alarms and 3 dashboards are free-tier; SNS email is effectively free; log storage pennies. Teardown included.
> **Prerequisites:** Labs 03–04 — Terraform manages your lake, and a Glue job produces curated data.

---

## 1. Environment Setup

### 1.1 Verify

```powershell
aws sts get-caller-identity
cd C:\sk04\terraform; terraform validate
aws glue get-job --job-name swifthaul-curate --query "Job.Name" 2>$null
```

**Expected output:** identity OK, `Success! The configuration is valid.`, and your Glue job's name (adjust to yours from Lab 04).

### 1.2 One new folder in your Terraform project

```powershell
New-Item C:\sk04\terraform\monitoring.tf -ItemType File
```

We keep all monitoring resources in one file — reviewable at a glance, exactly how ops teams like it.

### 1.3 Common problems

| Symptom | Fix |
|---|---|
| `terraform validate` fails before you start | Uncommitted experiments from Lab 03/04 — `terraform plan` and clean up first |
| No Glue job present | Re-apply Lab 04; this lab attaches monitoring *to* it |

---

## 2. Business Context

Your SwiftHaul pipeline now runs unattended: files land in S3, Glue curates, Redshift serves. The question every client asks next: **"How will we know when it breaks?"**

Without monitoring, the answer is "when the dashboard looks wrong and someone complains" — which for a logistics company means *ops planned tomorrow's routes on stale data*. The industry-standard answer on AWS is **CloudWatch**: every AWS service emits **logs** and **metrics** into it; you define **alarms** on the metrics; alarms notify humans through **SNS**; and a **dashboard** puts pipeline health on one screen.

**Who consumes this?**
- The **on-call engineer** — alarms are their pager.
- The **ops manager** — the dashboard is their morning glance.
- **You in the project** — the capstone *mandates* a Terraform-provisioned CloudWatch dashboard; this lab is its dress rehearsal.

**What failure looks like without it (real-world pattern):** a Glue job starts failing on a schema change Tuesday; nobody looks; Friday's month-end numbers are three days stale; the incident review's first finding is always the same — *"no alerting on job failure."*

---

## 3. Concept Explanation

### 3.1 The observability triad on AWS

| Piece | What it is | SwiftHaul example |
|---|---|---|
| **Logs** (CloudWatch Logs) | Timestamped text streams from services — your SK-01 log files, centralized | Glue job's Spark driver output in `/aws-glue/jobs/output` |
| **Metrics** | Numeric time series, per service, with dimensions | `glue.driver.aggregate.numFailedTasks`, S3 `NumberOfObjects`, Redshift `ComputeSeconds` |
| **Alarms** | A rule watching one metric (or math on several): threshold + evaluation periods → state change → action | "Glue job failed ≥ 1 time in 15 min → SNS email" |

Two supporting actors: **SNS** (Simple Notification Service — pub/sub topics that fan out alarm notifications to email/SMS/chat) and **EventBridge** (the event bus — reacts to *state changes* like "Glue job FAILED" even when no metric exists, or when you want event-driven wiring rather than threshold-watching).

### 3.2 Alarm design: the art of not crying wolf

An alarm that fires weekly for non-reasons gets muted — then the real failure is missed (the industry calls this **alert fatigue**, and it is the #1 monitoring failure mode). Principles:

- **Alarm on symptoms users feel** (job failed; data not fresh; error rate up), not on internals (CPU high) unless they predict symptoms.
- Every alarm must have an **action a human would take**. No action → it's a dashboard widget, not an alarm.
- **Treat missing data as a signal**: a job that *didn't run* emits no failure metric — `treat_missing_data = "breaching"` on the right alarms catches the silent-absence case (the cloud version of SK-02's "no successful run in 26 hours").

### 3.3 Dashboards as code

Clicking a dashboard together in the console works once — then it drifts, can't be code-reviewed, and vanishes if recreated. `aws_cloudwatch_dashboard` in Terraform stores the layout as JSON in your repo: reviewable, reproducible, and identical across environments. This is the exact deliverable the capstone requires.

---

## 4. Step-by-Step Implementation

### Step 1 — The notification channel (SNS)

**What:** In `monitoring.tf`:

```hcl
resource "aws_sns_topic" "pipeline_alerts" {
  name = "swifthaul-pipeline-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.pipeline_alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email          # add to variables.tf; set in terraform.tfvars
}
```

```powershell
terraform apply
```

**Why SNS in the middle** (instead of alarms→email directly): alarms fan into a *topic*; the topic decides delivery. Adding Slack or PagerDuty later means changing subscriptions, not touching thirty alarms.

**Verify:** check your inbox for *AWS Notification - Subscription Confirmation* and **click confirm** — unconfirmed subscriptions silently drop messages (a classic gotcha). Then prove the channel end-to-end:

```powershell
aws sns publish --topic-arn (terraform output -raw alerts_topic_arn) --message "SwiftHaul alert channel test"
```

**Expected output:** the test email arrives. *Never trust an alert path you haven't tested.*

### Step 2 — Alarm 1: Glue job failure (via EventBridge)

**What:** Glue job failures are *events*, and EventBridge → SNS is the crisp pattern:

```hcl
resource "aws_cloudwatch_event_rule" "glue_job_failed" {
  name        = "swifthaul-glue-job-failed"
  description = "Fires when any run of the curate job ends FAILED or TIMEOUT"
  event_pattern = jsonencode({
    source      = ["aws.glue"]
    detail-type = ["Glue Job State Change"]
    detail = {
      jobName = ["swifthaul-curate"]
      state   = ["FAILED", "TIMEOUT", "ERROR"]
    }
  })
}

resource "aws_cloudwatch_event_target" "glue_failed_to_sns" {
  rule = aws_cloudwatch_event_rule.glue_job_failed.name
  arn  = aws_sns_topic.pipeline_alerts.arn
}

resource "aws_sns_topic_policy" "allow_eventbridge" {
  arn    = aws_sns_topic.pipeline_alerts.arn
  policy = data.aws_iam_policy_document.sns_allow_events.json
}

data "aws_iam_policy_document" "sns_allow_events" {
  statement {
    effect    = "Allow"
    actions   = ["SNS:Publish"]
    resources = [aws_sns_topic.pipeline_alerts.arn]
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
  }
}
```

**Why the topic policy:** EventBridge is a *service* publishing to *your* topic — without the resource policy, delivery fails silently (no error surfaces anywhere; you just never get mail). This is the least-known, most-bitten step in the pattern.

**Apply, then verify by breaking it:** edit your Glue job to reference a nonexistent S3 path (console → job → edit script — or temporarily rename the source prefix), run it, and wait for the failure email. **Revert afterwards.** You have now rehearsed the exact incident this alarm exists for.

### Step 3 — Alarm 2: data freshness (metric alarm with missing-data handling)

**What:** "Did curated data actually get written today?" Use S3 object-count change... but the honest, robust freshness signal is a **custom metric your pipeline emits on success**:

```hcl
resource "aws_cloudwatch_metric_alarm" "curated_data_stale" {
  alarm_name          = "swifthaul-curated-data-stale"
  namespace           = "SwiftHaul/Pipeline"
  metric_name         = "CuratedLoadSuccess"
  statistic           = "Sum"
  period              = 86400          # 1 day
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "LessThanThreshold"
  treat_missing_data  = "breaching"    # no metric = no run = ALARM
  alarm_description   = "No successful curated load in 24h - ops dashboard is stale"
  alarm_actions       = [aws_sns_topic.pipeline_alerts.arn]
  ok_actions          = [aws_sns_topic.pipeline_alerts.arn]
}
```

And at the *end of your Glue job script* (success path only), emit the heartbeat:

```python
import boto3
boto3.client("cloudwatch").put_metric_data(
    Namespace="SwiftHaul/Pipeline",
    MetricData=[{"MetricName": "CuratedLoadSuccess", "Value": 1}],
)
```

(The Glue job's IAM role needs `cloudwatch:PutMetricData` — add it to the role policy in your Terraform from Lab 04.)

**Why this design:** `treat_missing_data = "breaching"` is the whole trick — the alarm catches *the job that never ran* (scheduler broke, trigger deleted), which failure-alarms structurally cannot see. Success-heartbeat + missing-data-breaches is the canonical freshness pattern, and interviewers love it. `ok_actions` sends the "recovered" email so on-call knows when to stand down.

**Verify:** run the Glue job; within a few minutes the metric appears (CloudWatch → Metrics → SwiftHaul/Pipeline). The alarm starts in `INSUFFICIENT_DATA`, moves to `OK` after the first datapoint-day. (For lab purposes, temporarily set `period = 3600` to watch the full cycle faster, then restore.)

### Step 4 — Alarm 3: cost anomaly guard

**What:** watchdog on Glue DPU consumption — the "runaway job" guard:

```hcl
resource "aws_cloudwatch_metric_alarm" "glue_runtime_excessive" {
  alarm_name          = "swifthaul-glue-runtime-excessive"
  namespace           = "Glue"
  metric_name         = "glue.driver.aggregate.elapsedTime"
  dimensions = {
    JobName  = "swifthaul-curate"
    JobRunId = "ALL"
    Type     = "gauge"
  }
  statistic           = "Maximum"
  period              = 300
  evaluation_periods  = 1
  threshold           = 900000         # 15 min in ms — 3x normal for this job
  comparison_operator = "GreaterThanThreshold"
  treat_missing_data  = "notBreaching"
  alarm_description   = "Curate job running 3x longer than normal - possible data explosion or hang"
  alarm_actions       = [aws_sns_topic.pipeline_alerts.arn]
}
```

**Why threshold = 3× normal:** runtime alarms are set from a **baseline** (you know the job takes ~5 min from Lab 04 runs), not from guesses. Document the baseline next to the threshold — future engineers must know *why* 15 minutes.

### Step 5 — The dashboard, as code

**What:** the capstone's mandated artifact — one screen of pipeline health:

```hcl
resource "aws_cloudwatch_dashboard" "pipeline" {
  dashboard_name = "swifthaul-pipeline-health"
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric", x = 0, y = 0, width = 12, height = 6
        properties = {
          title  = "Curated load heartbeat (daily successes)"
          region = var.aws_region
          stat   = "Sum", period = 86400
          metrics = [["SwiftHaul/Pipeline", "CuratedLoadSuccess"]]
        }
      },
      {
        type = "metric", x = 12, y = 0, width = 12, height = 6
        properties = {
          title  = "Glue job elapsed time (ms)"
          region = var.aws_region
          stat   = "Maximum", period = 300
          metrics = [["Glue", "glue.driver.aggregate.elapsedTime",
                      "JobName", "swifthaul-curate", "JobRunId", "ALL", "Type", "gauge"]]
        }
      },
      {
        type = "alarm", x = 0, y = 6, width = 24, height = 3
        properties = {
          title  = "Pipeline alarms"
          alarms = [
            aws_cloudwatch_metric_alarm.curated_data_stale.arn,
            aws_cloudwatch_metric_alarm.glue_runtime_excessive.arn,
          ]
        }
      },
      {
        type = "log", x = 0, y = 9, width = 24, height = 6
        properties = {
          title  = "Recent Glue errors"
          region = var.aws_region
          query  = "SOURCE '/aws-glue/jobs/error' | fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20"
        }
      }
    ]
  })
}
```

```powershell
terraform apply
```

**Why these four widgets:** heartbeat (is data flowing?), runtime trend (is it healthy?), alarm status strip (anything red?), recent errors (what's the story?). That's a complete 30-second morning health check — resist the urge to add fifteen more widgets; dashboards die of clutter.

**Verify:** console → CloudWatch → Dashboards → `swifthaul-pipeline-health`. Run the Glue job and watch the heartbeat widget update. Then the reproducibility proof: `terraform destroy -target aws_cloudwatch_dashboard.pipeline`, then `terraform apply` — the dashboard resurrects *identically*. Click-built dashboards can't do that; this is your talking point for the capstone review.

### Step 6 — Teardown (partial — read first)

Alarms/dashboards within free tier cost nothing; **keep them if you're proceeding to the project this week** (the project extends them). Ending the module or pausing long-term:

```powershell
terraform destroy -target aws_cloudwatch_dashboard.pipeline `
                  -target aws_cloudwatch_metric_alarm.curated_data_stale `
                  -target aws_cloudwatch_metric_alarm.glue_runtime_excessive `
                  -target aws_cloudwatch_event_rule.glue_job_failed `
                  -target aws_sns_topic.pipeline_alerts
```

**Verify:** `aws cloudwatch describe-alarms --query "MetricAlarms[].AlarmName"` → empty; SNS topics list → empty. Because everything was Terraform-managed, teardown is one command and *complete* — the quiet payoff of infrastructure as code.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Notification channel as a topic, tested end-to-end** | Delivery pluggable; unconfirmed/silent-drop failure modes rehearsed away. |
| **Event-driven failure alerts (EventBridge)** | State changes caught immediately, without polling or thresholds. |
| **Success-heartbeat + `treat_missing_data = breaching`** | Catches the job that *never ran* — the failure mode metric alarms can't see. |
| **Baseline-derived thresholds, documented** | Alarms tuned to reality; no alert fatigue, no mystery numbers. |
| **Every alarm has a human action; symptoms over internals** | The discipline that keeps pagers meaningful. |
| **Dashboard as Terraform JSON** | Reviewable, reproducible, environment-portable — and demanded by the capstone. |
| **Break-it verification of every alert path** | An untested alarm is a decoration; you fired each one deliberately. |

---

## 6. Reflection

### What you learned
The CloudWatch observability triad wired to your real pipeline: SNS channel, EventBridge failure alerts, a freshness heartbeat alarm that treats silence as breach, a baseline-tuned runtime alarm, and a Terraform-provisioned dashboard — all torn down (or kept) deliberately.

### Why it matters
"How do you monitor your pipelines?" is asked in essentially every data engineering interview, and the capstone requires this dashboard. More deeply: monitoring is what converts "it worked when I built it" into "we operate this."

### Interview questions (with model answers)

1. **"How would you monitor a batch pipeline on AWS?"**
   Failure events via EventBridge→SNS; a success-heartbeat custom metric with a freshness alarm treating missing data as breaching; runtime/volume alarms off measured baselines; logs centralized in CloudWatch Logs; one dashboard-as-code for at-a-glance health.

2. **"How do you catch a job that didn't run at all?"**
   Failure alarms can't — no run, no failure event. Emit a success heartbeat metric and alarm on its *absence* (`treat_missing_data = breaching`, period = expected cadence).

3. **"Logs vs metrics vs alarms?"**
   Logs: detailed event text for diagnosis. Metrics: numeric time series for trends/thresholds. Alarms: rules on metrics that change state and trigger actions. Diagnose in logs, detect with metrics+alarms.

4. **"How do you prevent alert fatigue?"**
   Alarm only on user-felt symptoms with a defined human action; thresholds from baselines; missing-data semantics chosen deliberately; OK-notifications on recovery; periodic pruning of alarms nobody acts on.

5. **"Why put a dashboard in Terraform?"**
   Code review, reproducibility across environments/accounts, drift-free, disaster-recoverable. Demo: destroy and re-apply, byte-identical.

6. **"EventBridge vs CloudWatch alarm — when each?"**
   Discrete state changes (job FAILED, object created) → EventBridge. Continuous quantities crossing thresholds (runtime, error rate, freshness) → metric alarms. Most pipelines need both.

### Common interview traps
- Only mentioning failure alerts — the *silent absence* case (heartbeat pattern) is what separates candidates.
- "I'd monitor CPU/memory" for a data pipeline — symptoms first: freshness, failures, volume, duration.
- Untested alert paths. Say the sentence: "and I verify by deliberately failing the job once."

### Key takeaways
1. Detect with metrics/events, diagnose with logs, notify through one tested channel.
2. Silence is a failure mode — heartbeat + missing-data-breaches.
3. Thresholds come from baselines you measured and documented.
4. Dashboards are code. Everything you built here re-applies in the capstone, enlarged.

**Next:** the [SK-04 capstone project](../Project/01_Business_Scenario.md) — the full SwiftHaul engagement: lake, warehouse, Terraform, and both mandated dashboards (this CloudWatch one, plus Power BI on the curated data).
