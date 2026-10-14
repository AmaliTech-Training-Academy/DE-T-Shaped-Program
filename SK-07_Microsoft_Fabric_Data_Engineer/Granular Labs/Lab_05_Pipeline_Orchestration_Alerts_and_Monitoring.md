# Lab 05 — Pipeline Orchestration, Alerts and Monitoring

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Finish `MedVista_Daily_Ingestion`: chain the Silver and Gold notebooks, add a failure path that notifies Teams and still marks the run as failed, rebuild Silver and Gold from Bronze with the pipeline, prove re-runs are safe, and schedule the daily run.
> **Time:** 3–4 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab_04](Lab_04_Silver_to_Gold_Star_Schema_and_Delta_Maintenance.md) — notebooks `00_land_to_bronze`, `01_bronze_to_silver`, `02_silver_to_gold` all work interactively; the pipeline from Lab_02 runs Set variable → 3 Copies → Bronze load
> **Guided lab coverage:** Part 2 Step 2.2 (complete pipeline, error handling, Teams activity) and Part 4 Step 4.2 (three-day run, re-run, failure test, schedule)
> **Cost:** $0 on the trial. **Turn the schedule off at the end of this lab** — scheduled runs fail by design once the sample dates are past, and each failed run still consumes capacity.

---

## Learning objectives

By the end of this lab you can:

- Pass a pipeline variable into notebook parameters and chain notebooks with dependency conditions.
- Explain how dependency conditions combine (OR on one upstream activity, AND across activities) and how Fabric decides a run's final status.
- Build the "generic error handling" pattern: one notification for any failure, plus a **Fail** activity so the run still shows **Failed**.
- Use the Monitor hub to inspect runs, activity inputs/outputs and notebook exit values.
- Rebuild Silver and Gold from Bronze, and prove re-runs don't change results.
- Schedule a pipeline — and turn the schedule off.

---

## 1. Environment Setup

Verify Lab_04:

1. Open `MedVista_Daily_Ingestion`: `Set_Run_Date` → three Copy activities → `Load_Landing_to_Bronze`.
2. In a notebook: `spark.table("medvista_gold.dbo.fact_encounters").count()` → `3119`.
3. Teams: you can post in at least one Teams channel or group chat with your work account. If your organisation doesn't use Teams, you'll use the **Office 365 Outlook** activity instead (Step 2).

---

## 2. Business Context

Last quarter MedVista found out a load had been failing for four days when the CMO asked why the dashboard still said Tuesday. A pipeline that fails silently is worse than no pipeline: people trust numbers that are stale.

The on-call engineer needs three things: a message when anything fails, a run that is visibly **Failed** in monitoring (not "Succeeded with a warning"), and the confidence to press **Re-run** without double-loading anything.

---

## 3. Concept Explanation

### 3.1 Dependency conditions

Each arrow between activities carries a condition: **On success**, **On fail**, **On completion** (success or fail) or **On skip** (the activity didn't run because something before it failed).

- Several conditions from the **same** upstream activity are **OR'd**: "on fail *or* on skip".
- Dependencies on **different** activities are **AND'd**: `Load_Landing_to_Bronze` waits for all three Copies to succeed.

### 3.2 One handler for every failure

When an activity fails, everything after it is **skipped**, all the way to the end of the chain. So attaching the handler to the **last** activity with **On fail + On skip** catches a failure anywhere:

```
Set_Run_Date → Copy ×3 → Load_Landing_to_Bronze → Transform_Bronze_to_Silver → Transform_Silver_to_Gold
                                                                                     │ on fail / on skip
                                                                                     ▼
                                                                              Notify_Failure (Teams)
                                                                                     │ on completion
                                                                                     ▼
                                                                              Fail_Pipeline (Fail)
```

### 3.3 Why the Fail activity

Fabric decides a run's status from its **leaf** activities (if a leaf was skipped, it looks at its parent). If the notification succeeds and is the leaf, the run is marked **Succeeded** even though the data load failed — the "try/catch swallowed the error" problem. A **Fail** activity after the notification makes the run end **Failed** with your error code and message. Using **On completion** means the run fails even if the Teams message itself fails.

On a good day, `Notify_Failure` and `Fail_Pipeline` are skipped; Fabric looks past them to `Transform_Silver_to_Gold`, which succeeded, so the run is **Succeeded**.

### 3.4 Notifications

The **Teams** activity posts to a channel or group chat using a connection signed in as you (in production: a service account). The **Office 365 Outlook** activity sends an email. Both accept dynamic content. Don't build new alerts on the old Office 365 connector webhook URLs (`outlook.office.com/webhook/...`): Microsoft is retiring Office 365 connectors in Teams in favour of Workflows.

---

## 4. Step-by-Step Implementation

### Step 1 — Chain the Silver and Gold notebooks

1. Add a **Notebook** activity `Transform_Bronze_to_Silver`: notebook `01_bronze_to_silver`, base parameter `process_date` (String) = `@variables('runDate')`. Connect **On success** from `Load_Landing_to_Bronze`.
2. Add `Transform_Silver_to_Gold`: notebook `02_silver_to_gold`, same base parameter. Connect **On success** from `Transform_Bronze_to_Silver`.
3. **Save**.

✅ **Checkpoint:** the canvas reads left to right: Set variable → 3 parallel Copies → 3 notebooks in sequence.

### Step 2 — Add the failure path

1. Add a **Teams** activity `Notify_Failure`.
   - **Settings** → Connection → **New** → sign in with your work account.
   - **Post in**: Channel (pick a team and channel) or Group chat.
   - **Subject** (channels only): `MedVista pipeline FAILED`
   - **Message** → **View in expression builder**:

     ```
     @concat('MedVista daily load FAILED. Date: ', variables('runDate'), ' | Run ID: ', pipeline().RunId, ' | Open the Monitor hub to find the failed activity.')
     ```

   No Teams? Use **Office 365 Outlook** with the same text as the body and yourself as recipient.
2. From `Transform_Silver_to_Gold`, drag the red **On fail** handle to `Notify_Failure`, then the grey **On skip** handle to `Notify_Failure` as well.
3. Add a **Fail** activity `Fail_Pipeline`: Fail message `@concat('MedVista daily load failed for ', variables('runDate'))`, Error code `MEDVISTA_LOAD_FAILED`. Connect **On completion** from `Notify_Failure`.
4. **Save**. Open **View → JSON code** (`{}` icon) and find `Notify_Failure`'s `dependsOn`: one entry for `Transform_Silver_to_Gold` with `dependencyConditions: ["Failed", "Skipped"]`.

Compare your pipeline with the summary in the guided lab (Part 2, Step 2.2):

```json
{
  "name": "MedVista_Daily_Ingestion",
  "parameters": {
    "processDate": {"type": "string", "defaultValue": ""}
  },
  "variables": {
    "runDate": {"type": "String"}
  },
  "activities": [
    {
      "name": "Set_Run_Date",
      "type": "SetVariable",
      "variableName": "runDate",
      "value": "@if(empty(pipeline().parameters.processDate), formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd'), pipeline().parameters.processDate)"
    },
    {
      "name": "Copy_EHR_Encounters",
      "type": "Copy",
      "dependsOn": [{"activity": "Set_Run_Date", "dependencyConditions": ["Succeeded"]}],
      "source": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "filePathType": "File path",
        "folderPath": "source_drops/epic",
        "fileName": "@concat('encounters_', variables('runDate'), '.csv')",
        "fileFormat": "Binary"
      },
      "destination": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "folderPath": "@concat('landing/epic/dt=', variables('runDate'))",
        "fileFormat": "Binary"
      },
      "productionSource": "SFTP connection MedVista_EHR_SFTP, path exports/encounters_<date>.csv",
      "retry": 3,
      "retryIntervalInSeconds": 60
    },
    {
      "name": "Copy_HR_Schedules",
      "type": "Copy",
      "dependsOn": [{"activity": "Set_Run_Date", "dependencyConditions": ["Succeeded"]}],
      "source": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "filePathType": "File path",
        "folderPath": "source_drops/workday",
        "fileName": "@concat('schedules_', variables('runDate'), '.csv')",
        "fileFormat": "Binary"
      },
      "destination": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "folderPath": "@concat('landing/workday/dt=', variables('runDate'))",
        "fileFormat": "Binary"
      },
      "productionSource": "REST connection MedVista_Workday_API, relative URL /api/v1/schedules?date=<date>, pagination rule AbsoluteUrl = $.next_page_url",
      "retry": 3,
      "retryIntervalInSeconds": 60
    },
    {
      "name": "Copy_Supply_Orders",
      "type": "Copy",
      "dependsOn": [{"activity": "Set_Run_Date", "dependencyConditions": ["Succeeded"]}],
      "source": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "filePathType": "File path",
        "folderPath": "source_drops/sap",
        "fileName": "@concat('supply_orders_', variables('runDate'), '.csv')",
        "fileFormat": "Binary"
      },
      "destination": {
        "connection": "medvista_bronze (Lakehouse)",
        "rootFolder": "Files",
        "folderPath": "@concat('landing/sap/dt=', variables('runDate'))",
        "fileFormat": "Binary"
      },
      "productionSource": "OData connection MedVista_SAP_OData, path /sap/opu/odata/sap/SUPPLY_SRV/Orders, query $filter=OrderDate eq <date>",
      "retry": 3,
      "retryIntervalInSeconds": 60
    },
    {
      "name": "Load_Landing_to_Bronze",
      "type": "Notebook",
      "dependsOn": [
        {"activity": "Copy_EHR_Encounters", "dependencyConditions": ["Succeeded"]},
        {"activity": "Copy_HR_Schedules",   "dependencyConditions": ["Succeeded"]},
        {"activity": "Copy_Supply_Orders",  "dependencyConditions": ["Succeeded"]}
      ],
      "notebook": "00_land_to_bronze",
      "baseParameters": {"process_date": {"type": "string", "value": "@variables('runDate')"}}
    },
    {
      "name": "Transform_Bronze_to_Silver",
      "type": "Notebook",
      "dependsOn": [{"activity": "Load_Landing_to_Bronze", "dependencyConditions": ["Succeeded"]}],
      "notebook": "01_bronze_to_silver",
      "baseParameters": {"process_date": {"type": "string", "value": "@variables('runDate')"}}
    },
    {
      "name": "Transform_Silver_to_Gold",
      "type": "Notebook",
      "dependsOn": [{"activity": "Transform_Bronze_to_Silver", "dependencyConditions": ["Succeeded"]}],
      "notebook": "02_silver_to_gold",
      "baseParameters": {"process_date": {"type": "string", "value": "@variables('runDate')"}}
    },
    {
      "name": "Notify_Failure",
      "type": "Teams",
      "dependsOn": [
        {"activity": "Transform_Silver_to_Gold", "dependencyConditions": ["Failed", "Skipped"]}
      ],
      "postIn": "Channel (or Group chat)",
      "subject": "MedVista pipeline FAILED",
      "message": "@concat('MedVista daily load FAILED. Date: ', variables('runDate'), ' | Run ID: ', pipeline().RunId, ' | Open the Monitor hub to find the failed activity.')"
    },
    {
      "name": "Fail_Pipeline",
      "type": "Fail",
      "dependsOn": [{"activity": "Notify_Failure", "dependencyConditions": ["Completed"]}],
      "message": "@concat('MedVista daily load failed for ', variables('runDate'))",
      "errorCode": "MEDVISTA_LOAD_FAILED"
    }
  ]
}
```

✅ **Checkpoint:** your activities, names and dependency conditions match the summary.

### Step 3 — Rebuild Silver and Gold from Bronze

Bronze is the replayable source of truth, so you can rebuild everything downstream with the pipeline. In any notebook:

```python
for t in ["medvista_silver.dbo.silver_encounters", "medvista_silver.dbo.silver_staff_schedules",
          "medvista_silver.dbo.silver_supply_orders", "medvista_gold.dbo.fact_encounters",
          "medvista_gold.dbo.fact_supply_costs", "medvista_gold.dbo.dim_hospital",
          "medvista_gold.dbo.dim_date"]:
    spark.sql(f"DROP TABLE IF EXISTS {t}")
    print("dropped", t)
```

Stop the notebook session (pipeline notebook runs start their own sessions).

### Step 4 — Run three days through the whole pipeline

Run the pipeline with `processDate` = `2026-03-17`. Wait for it to finish, then `2026-03-18`, then `2026-03-19`.

**Expected output (each run):** all activities green except `Notify_Failure` and `Fail_Pipeline`, which are **Skipped**; the run status is **Succeeded**. Each run takes a few minutes, mostly Spark session start-up for the three notebooks.

Open **Monitor** (left navigation) → select a run → select `Transform_Silver_to_Gold` → **Output**. The notebook's exit value includes `fact_encounters_written`:

| processDate | `fact_encounters_written` | `readmission_rate` |
|---|---|---|
| 2026-03-17 | 1041 | 0.0576 |
| 2026-03-18 | 1173 | 0.0677 |
| 2026-03-19 | 1248 | 0.0789 |

These are the guided lab's numbers: this time each date went through Silver *and* Gold before the next date started.

✅ **Checkpoint:** `fact_encounters` = 3,119 rows, `fact_supply_costs` = 755, and the three exit values match the table.

### Step 5 — Prove a re-run is safe

Run again with `processDate` = `2026-03-19`.

**Expected output:** **Succeeded**, and the totals are unchanged: `fact_encounters` 3,119, `fact_supply_costs` 755, Bronze still 1,281 encounter rows for 2026-03-19. Every write in the chain is a `replaceWhere` or a MERGE.

✅ **Checkpoint:** row counts identical before and after the re-run.

### Step 6 — Prove the alert path

Run with `processDate` = `2026-03-20` — there are no source files for that date.

Each Copy retries 3 times with 60 seconds between attempts, so this takes about four minutes. (To test faster, temporarily set the three Copies' Retry to 0, then set it back.)

**Expected output:**

- The three Copy activities **Failed** ("path does not exist" / file not found).
- `Load_Landing_to_Bronze`, both transforms: **Skipped**.
- `Notify_Failure`: **Succeeded**, and the message appears in Teams with the date `2026-03-20` and the run ID.
- `Fail_Pipeline`: **Failed** with `MEDVISTA_LOAD_FAILED`.
- The run status in Monitor: **Failed**.

✅ **Checkpoint:** one Teams message, one Failed run whose error is your message and code — not a generic one.

Optional: make a *data* failure instead. Temporarily change assertion 4 in `02_silver_to_gold` to `readmit_rate < 0.05`, run for `2026-03-19`, watch `Transform_Silver_to_Gold` fail with the AssertionError text and the same alert fire. Change it back.

### Step 7 — Schedule it, then turn it off

1. Pipeline → **Schedule** → **On**, Repeat **Daily**, time **06:30**, your time zone, start today → **Apply**. Leave `processDate` empty: `Set_Run_Date` will use yesterday's date.
2. Note what will happen tomorrow: there are no files for yesterday's date, so the run fails and alerts. In production, that's exactly what you want when a source didn't deliver.
3. Turn the schedule **Off** now (or after you've seen one scheduled run fail).

✅ **Checkpoint:** the schedule shows **Off**.

### Step 8 — (Optional) Faster notebook chains

Each notebook activity starts its own Spark session. If your workspace allows **high concurrency for pipelines** (Workspace settings → Data Engineering/Science → High concurrency), give the three notebook activities the same **session tag** (activity **Settings** → **Advanced**) so they share one session. Compare run durations in Monitor before and after.

---

## 5. Production Engineering Practices

1. **Every pipeline has a failure path and a Fail activity.** Silence is the worst failure mode.
2. **Alert with context**: date, run ID, where to look. A message that says only "failed" wakes someone up without helping them.
3. **Retries for I/O, assertions for data.** Copy activities retry; notebooks fail fast and loudly.
4. **Rebuildability is a feature.** If Silver and Gold can be dropped and rebuilt from Bronze by the pipeline, schema changes and bug fixes become routine.
5. **Schedules are cost.** Every schedule is someone's capacity; document who owns it and turn it off when it's not needed.
6. **Use a service account (or workspace identity) for production connections**, not a personal sign-in that breaks when someone leaves.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Run shows **Succeeded** although a Copy failed | No Fail activity, or the alert isn't connected to the last activity | Add `Fail_Pipeline` on completion of the alert; connect On fail + On skip from `Transform_Silver_to_Gold` |
| Alert never fires when a Copy fails | Alert connected to an earlier activity with only On fail | Use On fail **and** On skip from the **last** activity |
| Alert fires on successful runs | Condition set to On completion instead of On fail/On skip | Fix the arrow types; check `dependsOn` in the JSON view |
| Notebook activity fails: parameter ignored, wrong date processed | Parameter cell not toggled in that notebook | Toggle parameter cell on cell 1 |
| Teams activity: connection or permission error | Tenant policy blocks it, or no access to that channel | Use Office 365 Outlook, or set the activity to **Deactivated** and rely on the Fail activity + Monitor |
| Day 2/3 counts differ from Step 4 | Silver/Gold not dropped first, or days run out of order | Repeat Step 3, then run 17, 18, 19 in order |
| Notebook activity waits a long time to start | Capacity busy (throttling) or session start-up | Check the Capacity Metrics app; stop interactive sessions |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"How do you send one alert for a failure anywhere in a sequential pipeline?"** — Attach the alert to the last activity with On fail and On skip; failures upstream cause skips downstream, and conditions on the same upstream activity are OR'd. Follow it with a Fail activity so the run's status is Failed.
2. **"Your pipeline failed at 03:00 and you fixed the bug. What do you do?"** — Re-run for the same date. Because every write is idempotent (replaceWhere / MERGE), re-running is safe; check row counts before and after.
3. **"Why does the pipeline compute the date once in a variable?"** — A run can start before midnight and finish after; computing `utcNow()` in several activities can load different days in one run.

### Common interview traps

- The try/catch pattern (alert on failure, nothing after it) makes failed runs look **Succeeded**.
- Multiple arrows from *different* activities into one alert = AND: it only fires if they *all* fail.

### Key takeaways

- On fail + On skip from the last activity → one handler for everything; Fail activity → honest run status.
- Bronze lets you drop and rebuild Silver/Gold with the pipeline.
- Re-runs are safe because every layer's writes are idempotent.
- Schedules cost capacity: turn them off when the lab is done.

---

## Next

[Lab_06 — Warehouse, Shortcuts and SQL Views](Lab_06_Warehouse_Shortcuts_and_SQL_Views.md): expose Gold to MedVista's analysts through OneLake shortcuts and T-SQL views in the analytics workspace.
