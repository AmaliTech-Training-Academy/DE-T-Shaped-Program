# Lab 03 — Bronze to Silver: PySpark Cleaning and Delta MERGE

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Build `01_bronze_to_silver`: de-duplicate, standardise and validate the MedVista extracts, compute a CMS readmission flag that looks across days, and upsert encounters with Delta `MERGE` so late discharges update earlier rows.
> **Time:** 4–5 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab_02](Lab_02_Pipeline_Ingestion_Copy_and_Bronze_Load.md) — Bronze holds three extract dates (1,070 / 1,202 / 1,281 encounters)
> **Guided lab coverage:** Part 3, Step 3.1 (complete Bronze → Silver notebook and expected counts)
> **Cost:** $0 on the trial. Stop the Spark session at the end.

---

## Learning objectives

By the end of this lab you can:

- Apply the Silver rules: de-duplicate on a business key, standardise codes, enforce validity, set a null policy.
- Explain why the order of transformations matters (validate *before* computing window features).
- Use a window function (`lag` over `partitionBy ... orderBy`) and explain why it needs history from earlier batches.
- Write a Delta `MERGE` with a conditional update, and read its insert/update metrics from `DESCRIBE HISTORY`.
- Use Delta time travel to show a row before and after a late update.
- Write idempotent partition-scoped overwrites for tables that don't need MERGE.

---

## 1. Environment Setup

Verify Lab_02 in any notebook:

```python
display(spark.sql("""SELECT _extract_date, COUNT(*) AS rows
                     FROM medvista_bronze.dbo.ehr_patient_encounters
                     GROUP BY _extract_date ORDER BY _extract_date"""))
```

**Expected:** 1070 / 1202 / 1281. If a date is missing, run the pipeline for it (Lab_02 Step 7).

Create notebook `01_bronze_to_silver`. In its Explorer, add **medvista_silver** (pin it as the default lakehouse) and also add **medvista_bronze**. Make cell 1 the parameter cell:

```python
process_date = "2026-03-17"
```

---

## 2. Business Context

The CMO's first question is the CMS **30-day readmission rate**: of all encounters, how many are a patient coming back within 30 days of their previous discharge? Getting it wrong has consequences — CMS penalises hospitals with excess readmissions, so an over-count costs money and an under-count is a compliance problem.

Bronze can't answer it yet. In Lab_00 you found duplicates, eleven spellings of five encounter types, impossible discharge times, negative charges, and 134 encounters re-sent a day later with a discharge time. Silver turns that into one trustworthy row per encounter.

---

## 3. Concept Explanation

### 3.1 The Silver rules, and fix vs reject

| Defect | Rule | Fix or reject? |
|---|---|---|
| Exact duplicate rows | `dropDuplicates(["encounter_id"])` | Fix (keep one) |
| `ER`, `ED`, `Emergency`… | Map to `EMERGENCY`, `INPATIENT`, `OUTPATIENT`, `OBSERVATION`; trim + upper anything else | Fix |
| Discharge before admission | Filter out | **Reject** — no safe correction exists |
| Negative charges | Clamp to 0 | Fix (data-entry error, documented policy) |
| Missing insurance | `UNKNOWN` | Fix (null policy: unknown is a category) |
| No discharge yet | Keep, `length_of_stay_hours` = null | Valid — patient still admitted |

Rejected rows are counted (`raw − clean`) so a sudden jump is visible in the run log.

### 3.2 Window functions and why the readmission flag needs history

```python
Window.partitionBy("patient_id").orderBy("admission_datetime")
F.lag("discharge_datetime").over(...)      # previous stay's discharge for the same patient
```

`lag` only sees rows in the DataFrame you give it. If you compute the flag on today's batch alone, a patient discharged *yesterday* and readmitted *today* is invisible — the previous stay is in Silver, not in today's batch. The notebook therefore unions today's rows with the last 60 days of Silver history **for today's patients only**, computes the flag, and keeps the flag only for today's rows.

It also validates *before* computing the window: a row with discharge before admission would otherwise become someone's "previous stay".

### 3.3 Delta MERGE

```python
(target.alias("t")
       .merge(source.alias("s"), "t.encounter_id = s.encounter_id")
       .whenMatchedUpdate(condition="s.discharge_datetime IS NOT NULL", set={...})
       .whenNotMatchedInsertAll()
       .execute())
```

- **Not matched** → a new encounter: insert it.
- **Matched, with a discharge time** → the late update: set discharge, length of stay, charges, readmission flag, and `_process_date` (so Lab_04's Gold load knows this row changed today).
- **Matched, still no discharge** → nothing to update.

MERGE requires at most **one** source row per key — which is why de-duplication comes first.

### 3.4 Partition-scoped overwrite for append-style tables

Schedules and supply orders don't get late updates, so they don't need MERGE. They use `mode("overwrite")` with `replaceWhere` on `_extract_date` — same idempotency as Bronze. The helper `overwrite_slice()` creates the table on the first run and uses `replaceWhere` afterwards.

---

## 4. Step-by-Step Implementation

Build the logic interactively for **2026-03-17** first, then paste the complete notebook (Step 6).

### Step 1 — Read one extract and count

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

BRONZE = "medvista_bronze.dbo"
SILVER = "medvista_silver.dbo"

bronze_enc = spark.table(f"{BRONZE}.ehr_patient_encounters").filter(F.col("_extract_date") == process_date)
print(bronze_enc.count(), bronze_enc.dropDuplicates(["encounter_id"]).count())
```

**Expected output:** `1070 1050` — 20 duplicate rows.

### Step 2 — Standardise encounter types

```python
std = (bronze_enc.dropDuplicates(["encounter_id"])
       .withColumn("encounter_type",
           F.when(F.upper(F.trim("encounter_type")).isin("ER", "ED", "EMERGENCY"), "EMERGENCY")
            .when(F.upper(F.trim("encounter_type")).isin("IP", "INPATIENT"), "INPATIENT")
            .when(F.upper(F.trim("encounter_type")).isin("OP", "OUTPATIENT"), "OUTPATIENT")
            .when(F.upper(F.trim("encounter_type")).isin("OBS", "OBSERVATION"), "OBSERVATION")
            .otherwise(F.upper(F.trim("encounter_type")))))
display(std.groupBy("encounter_type").count().orderBy("encounter_type"))
```

**Expected output:** five values — `EMERGENCY`, `INPATIENT`, `OBSERVATION`, `OUTPATIENT`, `TELEHEALTH` (the trim handles `" Telehealth "`).

### Step 3 — Validity, length of stay, null policy

```python
clean = (std
    .filter((F.col("admission_datetime") < F.col("discharge_datetime")) | F.col("discharge_datetime").isNull())
    .withColumn("length_of_stay_hours",
        F.when(F.col("discharge_datetime").isNotNull(),
               F.round((F.unix_timestamp("discharge_datetime") - F.unix_timestamp("admission_datetime")) / 3600.0, 2)))
    .withColumn("insurance_type", F.coalesce("insurance_type", F.lit("UNKNOWN")))
    .withColumn("total_charges", F.when(F.col("total_charges") < 0, 0.0).otherwise(F.col("total_charges"))))
print(clean.count())
display(clean.groupBy("encounter_type").count().orderBy("encounter_type"))
```

**Expected output:** `1041` (9 impossible rows rejected), and EMERGENCY 383, INPATIENT 233, OBSERVATION 93, OUTPATIENT 301, TELEHEALTH 31.

✅ **Checkpoint:** raw 1,070 → de-duplicated 1,050 → clean 1,041, and you can say which rule removed each group.

### Step 4 — The readmission window on one day

```python
by_patient = Window.partitionBy("patient_id").orderBy("admission_datetime")
flags = (clean
    .withColumn("prev_discharge", F.lag("discharge_datetime").over(by_patient))
    .withColumn("is_readmission",
        F.coalesce(F.col("admission_datetime") <= F.col("prev_discharge") + F.expr("INTERVAL 30 DAYS"),
                   F.lit(False))))
print(flags.filter("is_readmission").count())
display(flags.filter("is_readmission")
        .select("patient_id", "prev_discharge", "admission_datetime", "encounter_type").limit(5))
```

**Expected output:** `60` — on day one, every readmission is a same-day return (there's no earlier history yet). The `coalesce(..., False)` turns "no previous stay" (null) into `False`.

Now think about 18 March: a patient discharged on the 17th and readmitted on the 18th has their previous stay in **Silver**, not in the 18 March batch. That's why the full notebook adds Silver history to the window input.

### Step 5 — MERGE, conceptually

Before running it, predict: on 18 March the batch contains 132 encounters from 17 March, now with discharge times. What should MERGE do with them? (Update discharge, length of stay, charges, flag and `_process_date`; insert nothing.) Keep your prediction for Step 7.

### Step 6 — The complete notebook, first day

Replace everything after the parameter cell with the guided lab's notebook:

```python
# Fabric Notebook: 01_bronze_to_silver
# Default lakehouse: medvista_silver. medvista_bronze is also added in the Explorer.
# Transforms one extract date of Bronze data into cleaned, conformed Silver tables.

# ── Parameters (this cell is marked as the PARAMETER cell) ──────────────────
process_date = "2026-03-17"

# ── Code ─────────────────────────────────────────────────────────────────────
import json
from pyspark.sql import functions as F
from pyspark.sql.window import Window
from pyspark.sql.utils import AnalysisException
from delta.tables import DeltaTable

BRONZE = "medvista_bronze.dbo"
SILVER = "medvista_silver.dbo"
print(f"Bronze -> Silver transformation for: {process_date}")
run_log = {"process_date": process_date, "tables": {}}


def table_exists(name):
    try:
        spark.sql(f"DESCRIBE TABLE {name}")
        return True
    except AnalysisException:
        return False


def overwrite_slice(df, table, predicate, partition_col):
    """Idempotent write: replace only the rows matching `predicate`.
    First run creates the table; later runs use replaceWhere."""
    writer = df.write.format("delta").mode("overwrite")
    if table_exists(table):
        writer = writer.option("replaceWhere", predicate)
    else:
        writer = writer.partitionBy(partition_col)
    writer.saveAsTable(table)


# ════════════════════════════════════════════════════════════════════
# TABLE 1: Patient Encounters  (MERGE - encounters get late updates)
# ════════════════════════════════════════════════════════════════════

bronze_enc = (
    spark.table(f"{BRONZE}.ehr_patient_encounters")
    .filter(F.col("_extract_date") == process_date)
)
raw_count = bronze_enc.count()

clean_enc = (
    bronze_enc
    # Deduplicate - the same encounter can be re-sent in one extract
    .dropDuplicates(["encounter_id"])

    # Standardise encounter types across EHR source spellings
    .withColumn("encounter_type",
        F.when(F.upper(F.trim("encounter_type")).isin("ER", "ED", "EMERGENCY"), "EMERGENCY")
         .when(F.upper(F.trim("encounter_type")).isin("IP", "INPATIENT"),       "INPATIENT")
         .when(F.upper(F.trim("encounter_type")).isin("OP", "OUTPATIENT"),      "OUTPATIENT")
         .when(F.upper(F.trim("encounter_type")).isin("OBS", "OBSERVATION"),    "OBSERVATION")
         .otherwise(F.upper(F.trim("encounter_type")))
    )

    # Validity filter FIRST, so invalid rows can't influence the readmission window
    .filter(
        (F.col("admission_datetime") < F.col("discharge_datetime")) |
        F.col("discharge_datetime").isNull()          # still admitted - valid
    )

    # Length of stay - only when discharged
    .withColumn("length_of_stay_hours",
        F.when(F.col("discharge_datetime").isNotNull(),
               F.round((F.unix_timestamp("discharge_datetime") -
                        F.unix_timestamp("admission_datetime")) / 3600.0, 2)))

    # Null policy: unknown insurance is a category, not a null
    .withColumn("insurance_type", F.coalesce(F.col("insurance_type"), F.lit("UNKNOWN")))
    # Validity: charges can't be negative (data-entry error -> 0)
    .withColumn("total_charges",
        F.when(F.col("total_charges") < 0, 0.0).otherwise(F.col("total_charges")))
)

# ── CMS readmission flag: admitted within 30 days of the SAME patient's previous
# discharge. The previous stay is usually in an EARLIER extract, so look back into
# Silver history for today's patients - a per-batch window would miss most of them.
keys = ["encounter_id", "patient_id", "admission_datetime", "discharge_datetime"]
window_input = clean_enc.select(*keys)
if table_exists(f"{SILVER}.silver_encounters"):
    history = (
        spark.table(f"{SILVER}.silver_encounters")
        .filter(F.col("admission_datetime") >=
                F.date_sub(F.lit(process_date).cast("date"), 60))
        .select(*keys)
        .join(clean_enc.select("patient_id").distinct(), "patient_id", "left_semi")
        .join(clean_enc.select("encounter_id"), "encounter_id", "left_anti")
    )
    window_input = window_input.unionByName(history)

by_patient = Window.partitionBy("patient_id").orderBy("admission_datetime")
readmission_flags = (
    window_input
    .withColumn("prev_discharge", F.lag("discharge_datetime").over(by_patient))
    .withColumn("is_readmission",
        F.coalesce(F.col("admission_datetime") <=
                   F.col("prev_discharge") + F.expr("INTERVAL 30 DAYS"), F.lit(False)))
    .select("encounter_id", "is_readmission")
)

silver_enc = (
    clean_enc.join(readmission_flags, "encounter_id", "left")
    .withColumn("_processed_at", F.current_timestamp())
    .withColumn("_process_date", F.lit(process_date).cast("date"))   # last batch that touched the row
    .drop("_source_system", "_ingested_at", "_file_name", "_extract_date")
)

clean_count = silver_enc.count()
reject_count = raw_count - clean_count      # duplicates + invalid rows

if table_exists(f"{SILVER}.silver_encounters"):
    (DeltaTable.forName(spark, f"{SILVER}.silver_encounters").alias("t")
     .merge(silver_enc.alias("s"), "t.encounter_id = s.encounter_id")
     .whenMatchedUpdate(
         condition="s.discharge_datetime IS NOT NULL",
         set={"discharge_datetime":   "s.discharge_datetime",
              "length_of_stay_hours": "s.length_of_stay_hours",
              "total_charges":        "s.total_charges",
              "is_readmission":       "s.is_readmission",
              "_processed_at":        "s._processed_at",
              "_process_date":        "s._process_date"})   # so Gold picks the update up
     .whenNotMatchedInsertAll()
     .execute())
else:
    (silver_enc.write.format("delta").mode("overwrite")
     .partitionBy("encounter_type")
     .saveAsTable(f"{SILVER}.silver_encounters"))

run_log["tables"]["silver_encounters"] = {
    "raw": raw_count, "clean": clean_count, "rejected": reject_count}
print(f"  Encounters - raw: {raw_count:,} | clean: {clean_count:,} | rejected: {reject_count:,}")


# ════════════════════════════════════════════════════════════════════
# TABLE 2: Staff Schedules  (partition-scoped overwrite)
# ════════════════════════════════════════════════════════════════════

silver_sched = (
    spark.table(f"{BRONZE}.hr_staff_schedules")
    .filter(F.col("_extract_date") == process_date)
    .dropDuplicates(["schedule_id"])
    .withColumn("shift_hours",
        F.round((F.unix_timestamp("shift_end") - F.unix_timestamp("shift_start")) / 3600.0, 2))
    # \b = word boundary. Without it "do" matches "EnDOscopy" and "rn" matches "inteRN".
    .withColumn("role_category",
        F.when(F.col("role").rlike(r"(?i)\b(physician|doctor|md|do|hospitalist|intern)\b"), "PHYSICIAN")
         .when(F.col("role").rlike(r"(?i)\b(nurse|rn|lpn|np)\b"),                         "NURSING")
         .when(F.col("role").rlike(r"(?i)\b(tech|technician|aide|assistant)\b"),          "SUPPORT")
         .otherwise("OTHER"))
    .withColumn("is_abnormal_shift", (F.col("shift_hours") > 16) | (F.col("shift_hours") < 2))
    .withColumn("_processed_at", F.current_timestamp())
    .drop("_source_system", "_ingested_at", "_file_name")
)
overwrite_slice(silver_sched, f"{SILVER}.silver_staff_schedules",
                f"_extract_date = '{process_date}'", "_extract_date")
run_log["tables"]["silver_staff_schedules"] = {"written": silver_sched.count()}


# ════════════════════════════════════════════════════════════════════
# TABLE 3: Supply Orders  (partition-scoped overwrite)
# ════════════════════════════════════════════════════════════════════

silver_supply = (
    spark.table(f"{BRONZE}.supply_orders")
    .filter(F.col("_extract_date") == process_date)
    .dropDuplicates(["order_id"])
    .withColumn("category_group",
        F.when(F.col("category").rlike("(?i)pharma|drug|medication"), "PHARMACEUTICALS")
         .when(F.col("category").rlike("(?i)surgical|implant"),        "SURGICAL")
         .when(F.col("category").rlike("(?i)ppe|glove|mask|gown"),     "PPE")
         .when(F.col("category").rlike("(?i)lab|test|reagent"),        "LABORATORY")
         .otherwise("GENERAL"))
    # Derive total_cost if missing (prevents a null measure in Gold)
    .withColumn("total_cost",
        F.coalesce(F.col("total_cost"), F.col("quantity") * F.col("unit_cost")))
    .filter(F.col("total_cost") > 0)
    .withColumn("_processed_at", F.current_timestamp())
    .drop("_source_system", "_ingested_at", "_file_name")
)
overwrite_slice(silver_supply, f"{SILVER}.silver_supply_orders",
                f"_extract_date = '{process_date}'", "_extract_date")
run_log["tables"]["silver_supply_orders"] = {"written": silver_supply.count()}

print(f"\nBronze -> Silver complete. Summary: {json.dumps(run_log, indent=2)}")
notebookutils.notebook.exit(json.dumps(run_log))
```

Run all with `process_date = "2026-03-17"`.

**Expected output:**

```
  Encounters - raw: 1,070 | clean: 1,041 | rejected: 29
```

and a summary with `silver_staff_schedules: 600` and `silver_supply_orders: 396`. On this first run `silver_encounters` doesn't exist yet, so the notebook creates it (partitioned by `encounter_type`) instead of merging.

✅ **Checkpoint:** `SELECT COUNT(*) FROM medvista_silver.dbo.silver_encounters` → `1041`.

### Step 7 — Days two and three: watch MERGE handle late updates

Change the parameter cell to `2026-03-18`, run all; then `2026-03-19`, run all. **Order matters**: the readmission look-back and the late-discharge updates depend on the earlier days already being in Silver.

**Expected output:**

```
  Encounters - raw: 1,202 | clean: 1,173 | rejected: 29     (2026-03-18)
  Encounters - raw: 1,281 | clean: 1,248 | rejected: 33     (2026-03-19)
```

Now read the MERGE metrics:

```python
display(spark.sql("DESCRIBE HISTORY medvista_silver.dbo.silver_encounters")
        .select("version", "operation",
                F.col("operationMetrics.numTargetRowsInserted").alias("inserted"),
                F.col("operationMetrics.numTargetRowsUpdated").alias("updated"))
        .orderBy("version"))
```

**Expected output:**

| version | operation | inserted | updated |
|---|---|---|---|
| 0 | CREATE TABLE AS SELECT | | |
| 1 | MERGE | 1041 | 132 |
| 2 | MERGE | 1037 | 211 |

(The first row's operation name can differ slightly between runtimes.)

✅ **Checkpoint:** your Step 5 prediction matches: 132 late discharges were updates, not new rows. `silver_encounters` now has **3,119** rows.

### Step 8 — Time-travel one late update

`E2026031700012` was admitted on 17 March at 02:30 and still in hospital when that day's extract ran:

```python
eid = "E2026031700012"
for v in (0, 1):
    display(spark.sql(f"""SELECT {v} AS version, encounter_id, encounter_type, discharge_datetime,
                                 length_of_stay_hours, _process_date
                          FROM medvista_silver.dbo.silver_encounters VERSION AS OF {v}
                          WHERE encounter_id = '{eid}'"""))
```

**Expected output:** version 0 → `INPATIENT`, discharge `null`, LOS `null`, `_process_date` 2026-03-17. Version 1 → discharge `2026-03-18 06:55:05`, LOS `28.42`, `_process_date` 2026-03-18.

✅ **Checkpoint:** you can explain why updating `_process_date` matters: Lab_04's Gold notebook picks up rows by `_process_date`, so without it the discharge would never reach Gold.

### Step 9 — Word boundaries in the role regex

The schedules section classifies free-text roles. Compare a naive pattern with the one in the notebook:

```python
roles = spark.table(f"{SILVER}.silver_staff_schedules").select("role").distinct()
naive = (F.when(F.col("role").rlike("(?i)physician|doctor|md|do"), "PHYSICIAN")
          .when(F.col("role").rlike("(?i)nurse|rn|lpn|np"), "NURSING")
          .when(F.col("role").rlike("(?i)tech|aide|assistant"), "SUPPORT").otherwise("OTHER"))
display(spark.table(f"{SILVER}.silver_staff_schedules")
        .select("role", "role_category").distinct()
        .withColumn("naive_category", naive)
        .filter(F.col("role_category") != F.col("naive_category")))
```

**Expected output:** two rows — `Endoscopy Tech` (naive: PHYSICIAN, because "en**do**scopy" contains "do") and `Medical Intern` (naive: NURSING, because "inte**rn**" contains "rn"). `\b...\b` matches whole words only.

Also check the abnormal-shift flag: `spark.table(f"{SILVER}.silver_staff_schedules").filter("is_abnormal_shift").groupBy("_extract_date").count()` → 16 / 18 / 19 (shifts over 16 hours or under 2).

### Step 10 — Final verification

```python
display(spark.sql(f"""
  SELECT 'encounters' AS t, COUNT(*) AS n FROM {SILVER}.silver_encounters UNION ALL
  SELECT 'still admitted',  COUNT(*) FROM {SILVER}.silver_encounters WHERE discharge_datetime IS NULL UNION ALL
  SELECT 'unknown insurance', COUNT(*) FROM {SILVER}.silver_encounters WHERE insurance_type = 'UNKNOWN' UNION ALL
  SELECT 'readmissions', COUNT(*) FROM {SILVER}.silver_encounters WHERE is_readmission UNION ALL
  SELECT 'schedules', COUNT(*) FROM {SILVER}.silver_staff_schedules UNION ALL
  SELECT 'supply orders', COUNT(*) FROM {SILVER}.silver_supply_orders"""))
```

**Expected output:** encounters 3,119 · still admitted 635 · unknown insurance 166 · readmissions 246 · schedules 1,800 · supply orders 1,182.

✅ **Checkpoint:** all six numbers match. Stop the session.

---

## 5. Production Engineering Practices

1. **Validate before you derive.** Window features computed over invalid rows contaminate valid ones.
2. **Look back across batches for stateful metrics.** Anything defined "relative to the previous event" (readmissions, churn, sessionisation) needs history, bounded to the relevant keys and time window to keep it cheap.
3. **De-duplicate before MERGE.** Delta refuses a MERGE where several source rows match one target row — fail fast instead of a random winner.
4. **Track "last touched by" on the row.** `_process_date` lets downstream loads be incremental *and* pick up updates.
5. **Log raw / clean / rejected counts every run.** The exit value carries them to the pipeline, where an alert can watch for spikes.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot perform Merge as multiple source rows matched and attempted to modify the same target row` | Source not de-duplicated on the merge key | Keep `dropDuplicates(["encounter_id"])` before the MERGE |
| `TABLE_OR_VIEW_NOT_FOUND medvista_bronze.dbo...` | Bronze lakehouse not added to the notebook, or no schemas | Add `medvista_bronze` in the Explorer; check the prefix |
| Day 2 shows 1,173 inserts and 0 updates | You deleted/recreated Silver between days, or ran days out of order | Drop `silver_encounters` and re-run 17, 18, 19 in order |
| Readmissions on day 2 lower than expected | History union missing (per-batch window only) | Use the full notebook; check `table_exists` returns True on day 2 |
| `Written data does not conform to partial table overwrite condition` | A row's `_extract_date` doesn't match the `replaceWhere` predicate | Filter Bronze on the same `_extract_date` you overwrite |
| `VERSION AS OF 1` shows no discharge | You ran day 2 more than once, or numbering differs | Check `DESCRIBE HISTORY` for the first MERGE version and use that |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"MERGE vs overwrite for Silver — how do you choose?"** — MERGE when source rows change after first arrival (late updates, corrections); partition-scoped overwrite when each extract is complete and final for its partition. Overwriting a table that receives late updates either loses them or forces full reloads.
2. **"Your readmission flag is computed per daily batch. What's wrong?"** — It misses readmissions where the previous stay was in an earlier batch, which is most of them. Union a bounded history window (here 60 days, today's patients only) before computing `lag`.
3. **"What does `whenMatchedUpdate(condition=...)` buy you over `whenMatchedUpdateAll()`?"** — Control over which changes are accepted. Here a re-sent row *without* a discharge must not overwrite a row that already has one.

### Common interview traps

- Calling `dropDuplicates()` "safe" when duplicates differ: it keeps an arbitrary row. If versions differ, rank them (e.g. latest `_ingested_at`) and keep the top one.
- Forgetting that `lag` returns null for the first row in each partition — always decide what null means (`False` here).

### Key takeaways

- Silver = one trustworthy row per business key, with documented fix-or-reject rules.
- Order: de-duplicate → standardise → validate → derive.
- Stateful flags need history from earlier batches.
- MERGE with a conditional update handles late-arriving data; `DESCRIBE HISTORY` proves what it did.

---

## Next

[Lab_04 — Silver to Gold: Star Schema and Delta Maintenance](Lab_04_Silver_to_Gold_Star_Schema_and_Delta_Maintenance.md): build `dim_date`, `dim_hospital` and the two fact tables, add quality assertions, and compact the facts with OPTIMIZE.
