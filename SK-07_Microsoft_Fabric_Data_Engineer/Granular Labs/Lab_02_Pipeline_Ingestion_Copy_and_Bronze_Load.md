# Lab 02 — Pipeline Ingestion: Copy Activities and the Bronze Load

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Build the first half of `MedVista_Daily_Ingestion`: a date parameter, three parallel Copy activities that land each day's source files, and the `00_land_to_bronze` notebook that loads them into the typed Bronze tables — idempotently.
> **Time:** 3–4 hours | **Difficulty:** Beginner–Intermediate | **Prerequisites:** [Lab_01](Lab_01_Lakehouses_OneLake_and_Bronze_Schemas.md) — three lakehouses, nine files under `medvista_bronze/Files/source_drops`, three empty Bronze tables
> **Guided lab coverage:** Part 2, Steps 2.1 (landing → Bronze notebook) and 2.2 (parameter, variable and Copy activities)
> **Cost:** $0 on the trial. Each Copy and notebook run consumes a little capacity; nothing in this lab runs on a schedule.

---

## Learning objectives

By the end of this lab you can:

- Create a Fabric pipeline with a parameter, a variable and dynamic-content expressions.
- Explain why a parameter's default can't be an expression, and use a **Set variable** activity instead.
- Configure Copy activities that build source and destination paths from the run date.
- Write a notebook with a **parameter cell** that a pipeline can override.
- Load CSVs into Delta with an explicit schema, lineage columns and `replaceWhere`, and prove the load is idempotent.

---

## 1. Environment Setup

Verify Lab_01:

1. `medvista_bronze` → **Files/source_drops** has `epic`, `workday`, `sap`, three CSVs each.
2. In any notebook attached to `medvista_bronze`:

   ```python
   display(spark.sql("SHOW TABLES IN medvista_bronze.dbo"))
   ```

   **Expected:** `ehr_patient_encounters`, `hr_staff_schedules`, `supply_orders`. Missing? Re-run `00_create_bronze_schemas` (Lab_01 Step 6).

---

## 2. Business Context

Every morning MedVista's three systems publish yesterday's data: Epic drops a CSV on SFTP, Workday serves a paginated REST API, SAP serves OData. Today an analyst downloads them by hand when they remember. The CMO wants this automated, retried when a source is flaky, and safe to re-run — because the first thing anyone does when a load fails is press **Run** again.

In production the three Copy activities point at those systems. In this lab, `Files/source_drops` stands in for them; only the Copy **source** setting would change:

| Activity | Lab source | Production source |
|---|---|---|
| Copy_EHR_Encounters | `Files/source_drops/epic` | SFTP connection to the Epic export server |
| Copy_HR_Schedules | `Files/source_drops/workday` | REST connection to Workday, pagination rule `AbsoluteUrl` = `$.next_page_url` |
| Copy_Supply_Orders | `Files/source_drops/sap` | OData connection to SAP `SUPPLY_SRV/Orders`, `$filter` on `OrderDate` |

---

## 3. Concept Explanation

### 3.1 Pipelines, activities and dynamic content

A **pipeline** (Data Factory in Fabric) is an orchestration graph of **activities**: Copy, Notebook, Set variable, Teams, Fail and many more. Most settings accept **dynamic content** — expressions that start with `@`:

| Expression | Meaning |
|---|---|
| `@pipeline().parameters.processDate` | Value passed in when the run starts |
| `@variables('runDate')` | Value set earlier in this run |
| `@concat('encounters_', variables('runDate'), '.csv')` | String building |
| `@formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')` | Yesterday (UTC) as text |

### 3.2 Parameters vs variables

A **parameter** is an input fixed for the whole run; its default must be a literal. A **variable** can be set during the run. So "default to yesterday unless told otherwise" is: parameter `processDate` defaults to `""`, and a **Set variable** activity computes

```
@if(empty(pipeline().parameters.processDate),
    formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd'),
    pipeline().parameters.processDate)
```

Every later activity uses `variables('runDate')`, never the parameter.

### 3.3 Land first, then load

The Copy activities copy files **byte for byte** (Binary format) into a dated landing folder: `Files/landing/epic/dt=2026-03-17/encounters_2026-03-17.csv`. A notebook then parses and types them. Two reasons:

- The landing folder is an immutable record of exactly what each source sent on each day, so you can always replay.
- Parsing happens once, in code, with the explicit Bronze schema — instead of being split between Copy mapping screens and notebooks.

### 3.4 Idempotency with `replaceWhere`

Appending the same extract twice doubles Bronze. Writing with

```python
(df.write.format("delta")
   .mode("overwrite")
   .option("replaceWhere", "_extract_date = '2026-03-17'")
   .saveAsTable("medvista_bronze.dbo.ehr_patient_encounters"))
```

replaces only that day's partition. Bronze still grows by one partition per day (append-only *across* days), but any single day can be re-run safely.

### 3.5 Parameter cells

In a Fabric notebook you mark one cell as the **parameter cell**. When a pipeline runs the notebook with base parameters, Fabric inserts a new cell right after it that overrides those variables. Interactive runs use the defaults you typed.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the pipeline, parameter and variable

1. `medvista-data-engineering` → **+ New item** → **Pipeline** → name `MedVista_Daily_Ingestion`.
2. Select an empty area of the canvas. In the bottom pane:
   - **Parameters** → **+ New** → Name `processDate`, Type `String`, Default value: leave **empty**.
   - **Variables** → **+ New** → Name `runDate`, Type `String`.
3. **Activities** → **Set variable** → name it `Set_Run_Date`. In **Settings**: Variable type **Pipeline variable**, Name `runDate`, Value → **Add dynamic content** → paste the expression from Section 3.2 → **OK**.
4. **Save**, then **Run** → leave `processDate` empty → **OK**.

**Expected output:** in the **Output** tab, `Set_Run_Date` **Succeeded**; open its output (the arrow icon) and the value is yesterday's date in `yyyy-MM-dd`.

5. Run again with `processDate` = `2026-03-17`.

✅ **Checkpoint:** the second run's output value is `2026-03-17`.

### Step 2 — Copy_EHR_Encounters

1. **Activities** → **Copy data** → **Add to canvas**. Name it `Copy_EHR_Encounters`. Drag the green **On success** handle from `Set_Run_Date` onto it.
2. **General**: Retry `3`, Retry interval `60` seconds.
3. **Source**:
   - Connection: `medvista_bronze` (pick it from the OneLake list)
   - Root folder: **Files**
   - File path type: **File path**
   - Directory: `source_drops/epic`
   - File name: **Add dynamic content** → `@concat('encounters_', variables('runDate'), '.csv')`
   - File format: **Binary**
4. **Destination**:
   - Connection: `medvista_bronze`, Root folder: **Files**
   - Directory: **Add dynamic content** → `@concat('landing/epic/dt=', variables('runDate'))`
   - File format: **Binary**

### Step 3 — Clone for Workday and SAP

Copy and paste `Copy_EHR_Encounters` twice (or right-click → **Duplicate**), connect each to `Set_Run_Date` with **On success**, and change:

| Activity | Source directory | Source file name | Destination directory |
|---|---|---|---|
| `Copy_HR_Schedules` | `source_drops/workday` | `@concat('schedules_', variables('runDate'), '.csv')` | `@concat('landing/workday/dt=', variables('runDate'))` |
| `Copy_Supply_Orders` | `source_drops/sap` | `@concat('supply_orders_', variables('runDate'), '.csv')` | `@concat('landing/sap/dt=', variables('runDate'))` |

The three Copy activities have no dependency on each other, so they run in parallel.

**Save**, **Run** with `processDate` = `2026-03-17`.

**Expected output:** all four activities **Succeeded**; each Copy shows 1 file read and written. In `medvista_bronze` → **Files** → refresh:

```
landing/epic/dt=2026-03-17/encounters_2026-03-17.csv
landing/workday/dt=2026-03-17/schedules_2026-03-17.csv
landing/sap/dt=2026-03-17/supply_orders_2026-03-17.csv
```

✅ **Checkpoint:** three landed files for 2026-03-17, byte-identical in size to the source drops (compare in **Properties**).

### Step 4 — Build the landing → Bronze notebook, piece by piece

Create notebook `00_land_to_bronze`, pin `medvista_bronze` as default lakehouse.

**Cell 1 — parameters.** Type the line below, then cell **…** → **Toggle parameter cell**. The cell gets a "Parameters" label.

```python
process_date = "2026-03-17"
```

**Cell 2 — read one file with the Bronze schema.** Try just the encounters first:

```python
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, TimestampType, DoubleType

ENC_SCHEMA = StructType([
    StructField("encounter_id", StringType()), StructField("patient_id", StringType()),
    StructField("encounter_type", StringType()), StructField("admission_datetime", TimestampType()),
    StructField("discharge_datetime", TimestampType()), StructField("department_id", StringType()),
    StructField("attending_physician", StringType()), StructField("primary_diagnosis", StringType()),
    StructField("secondary_diagnoses", StringType()), StructField("hospital_id", StringType()),
    StructField("insurance_type", StringType()), StructField("total_charges", DoubleType()),
])
path = f"Files/landing/epic/dt={process_date}/encounters_{process_date}.csv"
enc = (spark.read.format("csv").option("header", True).option("mode", "FAILFAST")
       .option("timestampFormat", "yyyy-MM-dd HH:mm:ss").schema(ENC_SCHEMA).load(path))
print(enc.count())
display(enc.select("encounter_id", "admission_datetime", "discharge_datetime",
                   "secondary_diagnoses", "insurance_type", "total_charges").limit(5))
```

**Expected output:** `1070`; timestamps shown as timestamps, empty discharge and insurance cells shown as `null`, `secondary_diagnoses` like `I10|E11.9`.

**Cell 3 — lineage columns.** Bronze records *where every row came from*:

```python
enc_lineage = (enc
    .withColumn("_source_system", F.lit("epic_ehr"))
    .withColumn("_ingested_at", F.current_timestamp())
    .withColumn("_file_name", F.col("_metadata.file_name"))     # Spark's hidden file-metadata column
    .withColumn("_extract_date", F.lit(process_date).cast("date"))
    .withColumn("secondary_diagnoses",
        F.coalesce(F.split("secondary_diagnoses", r"\|"), F.array().cast("array<string>"))))
display(enc_lineage.select("encounter_id", "secondary_diagnoses", "_file_name", "_extract_date").limit(3))
```

**Expected output:** `_file_name` = `encounters_2026-03-17.csv`; `secondary_diagnoses` is now an array, e.g. `["I10", "E11.9"]` (or `[]`).

✅ **Checkpoint:** you can explain each of the four lineage columns and why the pipe-delimited string becomes an array.

### Step 5 — The complete notebook

Replace the cells after the parameter cell with the full notebook from the guided lab (Step 2.1). Keep the parameter cell as cell 1.

```python
# Fabric Notebook: 00_land_to_bronze
# Default lakehouse: medvista_bronze (relative "Files/..." paths resolve against it).
# Loads one day's landed CSV files into the typed Bronze Delta tables.

# ── Parameters (this cell is marked as the PARAMETER cell) ──────────────────
process_date = "2026-03-17"

# ── Code ─────────────────────────────────────────────────────────────────────
import json
from pyspark.sql import functions as F
from pyspark.sql.types import (StructType, StructField, StringType, TimestampType,
                               DoubleType, IntegerType, DateType, BooleanType)

BRONZE = "medvista_bronze.dbo"
LANDING = "Files/landing"
print(f"Landing -> Bronze for extract date {process_date}")

ENC_SCHEMA = StructType([
    StructField("encounter_id", StringType()),
    StructField("patient_id", StringType()),
    StructField("encounter_type", StringType()),
    StructField("admission_datetime", TimestampType()),
    StructField("discharge_datetime", TimestampType()),
    StructField("department_id", StringType()),
    StructField("attending_physician", StringType()),
    StructField("primary_diagnosis", StringType()),
    StructField("secondary_diagnoses", StringType()),   # "I10|E11.9" in the file -> ARRAY in Bronze
    StructField("hospital_id", StringType()),
    StructField("insurance_type", StringType()),
    StructField("total_charges", DoubleType()),
])
SCHED_SCHEMA = StructType([
    StructField("schedule_id", StringType()),
    StructField("employee_id", StringType()),
    StructField("role", StringType()),
    StructField("department_id", StringType()),
    StructField("hospital_id", StringType()),
    StructField("shift_date", DateType()),
    StructField("shift_start", TimestampType()),
    StructField("shift_end", TimestampType()),
    StructField("is_overtime", BooleanType()),
])
SUPPLY_SCHEMA = StructType([
    StructField("order_id", StringType()),
    StructField("item_id", StringType()),
    StructField("item_name", StringType()),
    StructField("category", StringType()),
    StructField("quantity", IntegerType()),
    StructField("unit_cost", DoubleType()),
    StructField("total_cost", DoubleType()),
    StructField("department_id", StringType()),
    StructField("hospital_id", StringType()),
    StructField("order_date", DateType()),
    StructField("vendor_id", StringType()),
])


def load_to_bronze(folder, file_prefix, schema, table, source_system, extra=None):
    path = f"{LANDING}/{folder}/dt={process_date}/{file_prefix}_{process_date}.csv"
    df = (spark.read.format("csv")
          .option("header", True)
          .option("mode", "FAILFAST")                      # a malformed file stops the load
          .option("timestampFormat", "yyyy-MM-dd HH:mm:ss")
          .schema(schema)
          .load(path)
          .withColumn("_source_system", F.lit(source_system))
          .withColumn("_ingested_at", F.current_timestamp())
          .withColumn("_file_name", F.col("_metadata.file_name"))
          .withColumn("_extract_date", F.lit(process_date).cast("date")))
    if extra is not None:
        df = extra(df)
    target_cols = spark.table(table).columns               # align column order to the table
    # replaceWhere = idempotent: re-running a day replaces only that day's partition.
    (df.select(*target_cols)
       .write.format("delta").mode("overwrite")
       .option("replaceWhere", f"_extract_date = '{process_date}'")
       .saveAsTable(table))
    written = spark.table(table).filter(F.col("_extract_date") == process_date).count()
    print(f"  {table}: {written:,} rows for {process_date}")
    return written


def split_diagnoses(df):
    return df.withColumn(
        "secondary_diagnoses",
        F.coalesce(F.split("secondary_diagnoses", r"\|"), F.array().cast("array<string>")))


run_log = {
    "process_date": process_date,
    "ehr_patient_encounters": load_to_bronze("epic", "encounters", ENC_SCHEMA,
                                             f"{BRONZE}.ehr_patient_encounters", "epic_ehr",
                                             split_diagnoses),
    "hr_staff_schedules": load_to_bronze("workday", "schedules", SCHED_SCHEMA,
                                         f"{BRONZE}.hr_staff_schedules", "workday"),
    "supply_orders": load_to_bronze("sap", "supply_orders", SUPPLY_SCHEMA,
                                    f"{BRONZE}.supply_orders", "sap"),
}
print(json.dumps(run_log, indent=2))
notebookutils.notebook.exit(json.dumps(run_log))
```

Run all.

**Expected output:**

```
Landing -> Bronze for extract date 2026-03-17
  medvista_bronze.dbo.ehr_patient_encounters: 1,070 rows for 2026-03-17
  medvista_bronze.dbo.hr_staff_schedules: 606 rows for 2026-03-17
  medvista_bronze.dbo.supply_orders: 404 rows for 2026-03-17
```

followed by the JSON summary. Bronze keeps the duplicates and defects on purpose — Silver deals with them.

### Step 6 — Prove it's idempotent

Run the notebook again, unchanged.

**Expected output:** exactly the same three counts (not 2,140 / 1,212 / 808).

```python
display(spark.sql("DESCRIBE HISTORY medvista_bronze.dbo.ehr_patient_encounters")
        .select("version", "operation", "operationParameters"))
```

**Expected:** two `WRITE` operations whose parameters include `mode: Overwrite` and the predicate `_extract_date = '2026-03-17'`.

✅ **Checkpoint:** a second run doesn't change Bronze row counts, and `DESCRIBE HISTORY` shows why.

### Step 7 — Add the notebook to the pipeline and load the other two days

1. In the pipeline: **Activities** → **Notebook** → name `Load_Landing_to_Bronze`.
2. Connect **On success** from **each** of the three Copy activities to it (three arrows in = all three must succeed).
3. **Settings**: Workspace `medvista-data-engineering`, Notebook `00_land_to_bronze`. **Base parameters** → **+ New**: Name `process_date`, Type `String`, Value → dynamic content `@variables('runDate')`.
4. **Save**. Run with `processDate` = `2026-03-18`; when it succeeds, run with `2026-03-19`.

**Expected output:** each run: four green activities and a green notebook. In the notebook activity's output JSON you'll find the notebook's exit value (the JSON summary).

Verify in a notebook:

```python
display(spark.sql("""
    SELECT _extract_date, COUNT(*) AS rows
    FROM medvista_bronze.dbo.ehr_patient_encounters
    GROUP BY _extract_date ORDER BY _extract_date"""))
```

**Expected output:**

| _extract_date | rows |
|---|---|
| 2026-03-17 | 1070 |
| 2026-03-18 | 1202 |
| 2026-03-19 | 1281 |

`hr_staff_schedules` has 606 rows per date and `supply_orders` 404.

✅ **Checkpoint:** three partitions in each Bronze table, and the pipeline ran the notebook with the date you passed — not the `2026-03-17` default in the parameter cell.

Stop the notebook session when you're done.

---

## 5. Production Engineering Practices

1. **One date, computed once.** Every activity reads `variables('runDate')`. Mixing `utcNow()` calls across activities breaks runs that cross midnight.
2. **Retries on I/O, not on logic.** Copy activities get 3 retries (flaky networks); notebooks don't — a retried logic bug fails three times.
3. **Immutable landing, replayable Bronze.** Keep landed files; Bronze can be rebuilt from them.
4. **Idempotent writes everywhere.** `replaceWhere` per extract date in Bronze; MERGE and partition-scoped overwrites downstream (Labs 03–04).
5. **Fail at the door.** FAILFAST plus an explicit schema stops a malformed extract before it reaches Silver.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Copy fails with "file not found" / path doesn't exist | Wrong date, typo in directory, or `.csv` missing from the file-name expression | Open the activity input (Output tab) to see the evaluated path; compare with **Files** |
| Copy succeeds but nothing appears in `landing` | Destination root folder set to **Tables**, or the explorer isn't refreshed | Set root folder **Files**; refresh |
| Every pipeline run loads 2026-03-17 whatever date you pass | The first cell isn't marked as the parameter cell, so the pipeline's value is inserted *before* your hard-coded default and then overwritten | Toggle parameter cell on cell 1; re-run |
| `Malformed records are detected in record parsing` | A file doesn't match the schema (FAILFAST working as designed) | Find the bad row; if the source changed format, update the schema deliberately |
| `_metadata` cannot be resolved | Very old runtime | Use `F.input_file_name()` instead, or switch the workspace to Runtime 1.3 |
| Bronze row counts doubled | Wrote with `mode("append")` | Delete the day with `DELETE FROM ... WHERE _extract_date = '...'` (or re-run the notebook, which replaces it) and use `replaceWhere` |
| Notebook activity: "notebook not found" | Notebook in another workspace or renamed | Re-select it in the activity **Settings** |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"How do you make a daily pipeline default to yesterday but still support backfills?"** — Parameter with an empty default; Set variable with `if(empty(param), yesterday, param)`; every activity uses the variable.
2. **"Why land raw files before loading tables?"** — An immutable copy of what the source sent; replay without re-extracting; parsing logic in one place.
3. **"What makes a load idempotent?"** — Re-running it with the same inputs leaves the target in the same state. Here: overwrite only the partition for that extract date (`replaceWhere`); downstream, MERGE on business keys.

### Common interview traps

- "Bronze must always be append-only, so never overwrite." Append-only across extracts, yes. Re-loading the *same* extract must replace it, or retries create duplicates.
- Putting an expression in a parameter default and wondering why it arrives as literal text.

### Key takeaways

- Parameters are inputs; variables are computed. Use a Set variable for "default to yesterday".
- Copy activities land bytes; a notebook with an explicit schema types them.
- `replaceWhere` on `_extract_date` makes Bronze loads safely repeatable.
- Notebook parameter cells let one notebook serve interactive runs and pipelines.

---

## Next

[Lab_03 — Bronze to Silver: PySpark Cleaning and Delta MERGE](Lab_03_Bronze_to_Silver_PySpark_and_MERGE.md): de-duplicate, standardise and validate the data you just loaded, and handle late-arriving discharges with MERGE.
