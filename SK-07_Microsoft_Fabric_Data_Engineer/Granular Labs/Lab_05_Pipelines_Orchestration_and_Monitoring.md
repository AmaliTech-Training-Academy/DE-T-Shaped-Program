# Lab 05 — Data Pipelines: Orchestration, Scheduling, and Monitoring

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Estimated time:** 3.5–4.5 hours
> **Prerequisites:** Labs 02–04 — ingestion pipeline `pl_ingest_freshcart`, notebooks `01_bronze_to_silver` and `02_silver_to_gold`, and the warehouse layer all working.

---

## 1. Environment Setup

Verify the pieces this lab assembles:

1. `pl_ingest_freshcart` runs green (Lab 02).
2. `01_bronze_to_silver` runs top-to-bottom cold, parameter cell tagged (Lab 03 — re-check the tag: ⋯ menu on the cell → it should say *parameter cell*).
3. `02_silver_to_gold` runs top-to-bottom cold.
4. Know where the **Monitoring hub** is (left nav → Monitor) — it lists every item run in the workspace.

**Common problems:** notebooks fail cold-run because of out-of-order interactive state — fix the notebook (this is exactly why the discipline exists); parameter cell not tagged — pipeline overrides will silently not apply.

---

## 2. Business Context

FreshCart's exports land nightly at ~02:00. Right now, *you* are the scheduler: you click run on the ingestion pipeline, then the silver notebook, then gold. The client is paying for a platform, not for you clicking buttons at 2 a.m.

**Orchestration** turns your items into one automatic, ordered, supervised nightly run:

- **Ordering & dependency** — gold must not build from stale silver; silver must not run before landing completes.
- **Parameters** — "process the 2026-06-30 drop" flows from pipeline to notebooks (backfills and reruns become one parameter change).
- **Failure policy** — retries for the transient, alerts for the real, and *no downstream step runs on a failed upstream*.
- **Observability** — the Monitoring hub run history is your audit table; alerts are your pager.

This is SK-03's Airflow story with Fabric's tooling — deliberately so: orchestration concepts transfer across every tool; only the buttons move. **If this layer is weak:** the 4 a.m. silent failure → stale 8 a.m. dashboard → ops plans deliveries on yesterday's stock — the incident that ends platform trust.

---

## 3. Concept Explanation

### 3.1 Fabric Data Pipeline anatomy

A **Data pipeline** is Fabric's orchestrator (Data-Factory lineage): a canvas of **activities** connected by **conditions**:

| Piece | Meaning |
|---|---|
| **Activities** | Units of work: Copy data, **Notebook**, Invoke pipeline, Stored procedure, Teams/Outlook message, Web call… |
| **Dependency conditions** | On **Success** (green), **Failure** (red), **Completion** (either), **Skipped** — the arrows between activities |
| **Parameters & variables** | Pipeline-level inputs (e.g., `run_date`) passed into activities; expressions via the dynamic content editor |
| **Invoke pipeline** | Pipelines calling pipelines — how you compose ingest + transform into one master flow |

### 3.2 Failure semantics done right

Per-activity **retry** (count + interval) absorbs transient flakiness (capacity blips, brief throttling). The **On Failure path** handles real failures: typically an Outlook/Teams alert activity. Key subtlety: a pipeline shows *failed* only if a leaf activity fails with no failure-handler consuming it — so alert-then-fail requires the alert activity itself to end in a **Fail activity**, or the run reads "Succeeded" because the alert succeeded. (You will build this correctly in Step 4 — most tutorials get it wrong.)

### 3.3 Scheduling and event triggers

- **Schedules** — cron-like: nightly 03:00 after the 02:00 drop.
- **Event triggers (Reflex/Activator)** — run *when a file lands* in OneLake/Blob: the modern "don't poll, react" pattern. We schedule in this lab and note the event option for the project.

### 3.4 The Monitoring hub and run history

Every pipeline/notebook/dataflow run lands in **Monitor** with status, duration, and drill-in (per-activity timings; notebook cell output including your **reconciliation asserts** from Lab 03). Duration trends are your baselines — "gold took 9 minutes; it normally takes 3" is an incident *before* anyone sees wrong numbers. This hub + alerts = the CloudWatch of Lab SK-04, Fabric-flavored.

---

## 4. Step-by-Step Implementation

### Step 1 — The master pipeline skeleton

**What:** Workspace → **+ New item → Data pipeline** → `pl_freshcart_nightly`.

1. **Parameters** tab (canvas background → Parameters): add `run_date`, type String, default `2026-06-30`.
2. Add an **Invoke pipeline** activity → name `Ingest bronze` → invoked pipeline: `pl_ingest_freshcart`.
3. Add a **Notebook** activity → `Bronze to silver` → notebook `01_bronze_to_silver`. Under **Base parameters** add `run_date` → dynamic content → `@pipeline().parameters.run_date`.
4. Add a second **Notebook** activity → `Silver to gold` → `02_silver_to_gold`.
5. Wire: `Ingest bronze` —On Success→ `Bronze to silver` —On Success→ `Silver to gold`.

**Why Invoke-pipeline instead of rebuilding ingestion here:** composition — `pl_ingest_freshcart` stays independently testable, and the master pipeline reads like the architecture: land → refine → serve. (Airflow users: activities ≈ tasks, the canvas ≈ the DAG.)

**Verify:** validate button (toolbar) reports no errors.

### Step 2 — Retries where they belong

**What:** For each notebook activity → **Settings → Retry = 2, Retry interval = 120 s**. Leave the Invoke/Copy at 1 retry.

**Why 2×120s and not more:** notebook failures here are either transient capacity blips (one retry usually clears) or *real* (code/data — no number of retries fixes them, they just delay the alert). Bounded, spaced, few — the SK-01 retry philosophy verbatim. Document the numbers; "why 2?" must have an answer.

### Step 3 — Run it end to end

**What:** ▶ **Run** → confirm parameter `run_date` → watch the Output pane: three activities go `In progress → Succeeded` in order.

**Expected output:** total run of a few minutes; click each activity's glasses-icon → for notebooks, open the **snapshot** — you can see cell outputs, including your Lab 03 reconciliation print and assert. That snapshot *is* your run evidence; screenshot one for the capstone.

**Verify data actually moved:** SQL endpoint on `freshcart_gold` → `SELECT COUNT(*) FROM fct_sales;` — matches Lab 03's count (idempotent MERGE upstream means the rerun didn't inflate anything — the discipline pays off visibly here).

**Troubleshooting:**
| Symptom | Fix |
|---|---|
| Notebook activity can't find the notebook | It's in another workspace, or was renamed — re-select in activity settings |
| Parameter not reaching the notebook | Cell not tagged as parameter cell; name mismatch (`run_date` exactly) |
| First activity queued for minutes | Capacity concurrency — trial capacities queue; see it in Monitor hub |

### Step 4 — Failure path that actually fails

**What:**
1. Add an **Office 365 Outlook** activity (or **Teams** if your tenant allows) → name `Alert on failure` → To: you; Subject: dynamic content →

   ```
   @concat('FreshCart nightly FAILED — run ', pipeline().parameters.run_date)
   ```

   Body: include `@pipeline().RunId` and which stage, e.g. reference `@activity('Bronze to silver').Error.Message`.
2. Wire **On Failure** (red) arrows from *each* of the three activities into `Alert on failure`.
3. Add a **Fail activity** after the alert (On Success from the alert) → error message `Upstream failure — see alert email`, error code `500`.

**Why the Fail activity:** without it, "alert sent successfully" makes the whole *run* report Succeeded — the scheduler's history would show green on a night the data never loaded. Alert **and** fail: humans notified, history truthful, downstream (if any) blocked. This is the detail that separates a real failure design from a tutorial one.

**Verify by breaking it (the ritual):** in `01_bronze_to_silver`, temporarily add a first cell `raise RuntimeError("orchestration drill")`. Run the pipeline. **Expected:** `Bronze to silver` fails after its 2 retries (watch them in the output — attempt counts are visible), the email arrives with run ID and error, `Silver to gold` shows **Skipped**, the run shows **Failed**. Remove the drill cell. You have rehearsed the 4 a.m. incident on your own terms.

### Step 5 — Schedule it

**What:** Pipeline toolbar → **Schedule**: On; repeat **Daily**; time **03:00**; timezone: yours; Start date today. Save.

But the default `run_date` won't do for a schedule — make it dynamic for scheduled runs: in the notebook activity's base parameter, replace the static binding with:

```
@formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')
```

…or better: keep the pipeline parameter, and set *its* default to that expression (Parameters → default value → dynamic content). Manual runs can still override it — which is precisely your backfill mechanism: run the pipeline manually with `run_date = 2026-06-15` and only that drop reprocesses (idempotent MERGE absorbs any overlap).

**Why "yesterday":** the 03:00 run processes the drop that landed at 02:00, which carries the previous business day's date. Off-by-one date logic is among the most common scheduling bugs in the wild — reason it through once, write a comment, sleep forever after.

**Verify:** tomorrow, check Monitor hub for the 03:00 run (or set a one-off schedule 10 minutes ahead today and watch it fire, then set the real one).

### Step 6 — Monitoring, alerts, and the audit view

**What:**
1. **Monitor hub** (left nav) → filter to your workspace → you should see your manual runs, the drill failure, and (tomorrow) scheduled runs — status, duration, parameters per run. This is your SK-02 audit table, generated for free. Note durations: write your baseline down (e.g., "nightly ≈ 6 min").
2. **Duration/reliability watch:** open a run → per-activity durations. The habit: any activity at 3× baseline is investigated *even if green* (data explosion, capacity contention — you built this alarm in SK-04; here the first version is a human habit plus the hub's history, and the project asks you to document it as your monitoring strategy).
3. **Quarantine watch:** add one final **Notebook** activity `Quality check` (On Success after gold) with a tiny notebook that reads the quarantine growth for this run and **raises if above threshold**:

   ```python
   run_date = "override-me"  # parameter cell
   from pyspark.sql import functions as F
   q = spark.read.table("freshcart_silver.orders_quarantine").filter(F.col("run_date") == run_date)
   n = q.count()
   print(f"Quarantined this run: {n}")
   assert n < 500, f"QUALITY ALERT: {n} rows quarantined — investigate source data"
   ```

   Wire its On Failure into the same alert+fail branch.

**Why:** this completes the loop — the pipeline now fails (and emails you) not just when code breaks but when **data quality degrades**, exactly the "quarantine growth is a monitored metric" promise from Lab 03. Your platform now has: ordering, retries, truthful failures, alerting, scheduling, backfill, run audit, baselines, and a data-quality tripwire. That list *is* the production-readiness checklist for the capstone's orchestration section.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Composition via Invoke pipeline** | Ingest stays independently testable; the master reads like the architecture. |
| **Parameters flowing pipeline → notebook** | One mechanism for nightly runs, reruns, and backfills. |
| **Bounded, spaced retries — documented** | Transient absorbed; real failures alert fast; numbers have reasons. |
| **Alert + Fail activity pattern** | Humans notified *and* run history truthful — the subtle half most designs miss. |
| **Fire-drill verification** | You watched retries, skip-propagation, the email, and the failed status before trusting the schedule. |
| **Dynamic date defaults with off-by-one reasoning** | The most common scheduler bug, defused deliberately. |
| **Monitor hub as audit + baselines** | Run history, durations, parameters per run — free; anomaly = 3× baseline habit. |
| **Data-quality tripwire as a pipeline stage** | Quality degradation fails the run like any other failure — gates all the way down. |

---

## 6. Reflection

### What you learned
Composing ingestion and notebooks into one parameterized master pipeline with correct dependency, retry, and failure semantics; scheduling it with sane date logic; backfilling via parameter override; and monitoring through run history, baselines, alert emails, and a quarantine tripwire.

### Why it matters
Orchestration is where a collection of working pieces becomes a *platform someone can be on-call for*. Every concept here (DAG ordering, retries, alert-and-fail, backfill, baselines) transfers verbatim to Airflow, ADF, and every other orchestrator — you've now done it in two stacks (SK-03, SK-07), which is exactly the "concepts over tools" story to tell in interviews.

### Interview questions (with model answers)

1. **"How would you orchestrate a nightly medallion refresh in Fabric?"**
   A master Data pipeline: Invoke-pipeline for ingestion → notebook activities for silver and gold, chained On Success; run_date parameter flowing to tagged notebook parameter cells; retries 2×120s; On Failure → alert activity → Fail activity; daily schedule with a yesterday-date default; Monitor hub for history.

2. **"Your alert emails work but the pipeline shows Succeeded on failures. Why?"**
   The failure handler consumed the failure: the alert activity succeeded, so the run did. Append a Fail activity after the alert so the run status stays truthful and downstream/schedule semantics behave.

3. **"How do you backfill a missed day?"**
   Manual run with the run_date parameter overridden. Safe because the notebooks are idempotent (MERGE / scoped writes) — reprocessing overlaps changes nothing.

4. **"What do you monitor beyond success/failure?"**
   Duration vs baseline (3× = investigate even if green), data-quality signals (quarantine growth as a run-failing check), row-count reconciliations inside the jobs, and schedule freshness (did the 03:00 run happen at all).

5. **"Schedule triggers vs event triggers?"**
   Schedule: predictable cadence, simple, right when drops are time-reliable. Event (file-arrival): reacts to reality, no polling gap, right when arrival times vary — mention Fabric's Activator/Reflex for OneLake events.

6. **"Why must the notebooks be idempotent before you dare schedule them?"**
   Schedulers retry and humans rerun; without idempotency every recovery action duplicates data — the cure becomes the incident. Orchestration multiplies whatever safety properties the jobs have, including their absence.

### Common interview traps
- Retrying everything many times — bounded/spaced/few, and *why* (real failures just get slower alerts).
- No answer for "how do you know it ran?" — schedule freshness and run history are half of monitoring.
- Backfill = "edit the notebook date and run it" — parameters exist precisely so code never changes for a rerun.

### Key takeaways
1. The master pipeline is the architecture diagram, executable.
2. Alert **and** fail — truthful history or nothing.
3. Backfill = parameter override + idempotent jobs. No code edits, ever.
4. Baselines and the quarantine tripwire watch the two things success-status can't: slowness and quality decay.

**Next:** [Lab 06 — Semantic Models and Direct Lake Power BI](Lab_06_Semantic_Models_and_Direct_Lake_Power_BI.md) (already in your Labs folder) — putting the gold layer in front of FreshCart's decision-makers, then the [capstone](../Project/01_Business_Scenario.md).
