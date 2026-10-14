# Lab 01 — Lakehouses, OneLake and Bronze Schemas

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Create the three medallion lakehouses, upload the MedVista source drops to OneLake, run your first Fabric notebook, and create the typed Bronze Delta tables.
> **Time:** 2–3 hours | **Difficulty:** Beginner | **Prerequisites:** [Lab_00](Lab_00_Environment_Setup.md) — trial active, `medvista-data-engineering` workspace on the trial, nine CSVs in `C:\DataEngineering\sk07\data\source_drops`
> **Guided lab coverage:** Part 1, Steps 1.1 (lakehouses), 1.3 (upload) and 1.4 (Bronze schemas)
> **Cost:** $0 on the trial. A notebook's Spark session consumes capacity while it is open — stop it when you finish (Step 7).

---

## Learning objectives

By the end of this lab you can:

- Explain OneLake, a lakehouse, and the difference between its **Tables** and **Files** areas.
- Read a OneLake path and explain the `lakehouse.schema.table` naming used in Spark.
- Upload files to a lakehouse and read them with Spark.
- Explain why Bronze uses an explicit schema, and show what schema inference does to a messy column.
- Create partitioned Delta tables with `CREATE TABLE ... USING DELTA PARTITIONED BY`.

---

## 1. Environment Setup

Verify Lab_00 before you start:

1. Fabric home → **Workspaces** → `medvista-data-engineering` opens, and **+ New item** offers **Lakehouse** and **Notebook**.
2. Locally:

   ```powershell
   Get-ChildItem C:\DataEngineering\sk07\data\source_drops -Recurse -File | Select-Object Name, Length
   ```

   **Expected:** nine files — three `encounters_*`, three `schedules_*`, three `supply_orders_*`. Missing? Re-run `python generate_medvista_data.py` (Lab_00 Step 5).

Nothing new is installed in this lab.

---

## 2. Business Context

MedVista's EHR exports contain protected health information (PHI). The CMO wants analysts to self-serve, but the compliance officer wants raw PHI seen by as few people as possible, and wants a bad export to stay contained instead of reaching the board's dashboard.

The **medallion architecture** answers both:

| Layer | Lakehouse | Contents | Who reads it |
|---|---|---|---|
| Bronze | `medvista_bronze` | Raw extracts, typed but uncorrected, plus lineage columns | Data engineers only |
| Silver | `medvista_silver` | De-duplicated, standardised, validated | Engineers, data scientists |
| Gold | `medvista_gold` | Star schema built for reporting | Analysts (through Lab_06's Warehouse) |

Three lakehouses (rather than three folders in one) means three separate permission sets, and each layer can be rebuilt from the one below it.

---

## 3. Concept Explanation

### 3.1 OneLake and lakehouses

**OneLake** is the one data lake behind every Fabric tenant — think "OneDrive for data". Every workspace is a folder in it; every lakehouse is a folder in a workspace. Under the hood it's Azure Data Lake Storage, so any tool that speaks ADLS (`abfss://`) can read it.

A **lakehouse** is an item with two areas:

- **Files** — unmanaged: any file you put there (CSV, JSON, images). Spark reads it; nothing else interprets it.
- **Tables** — managed: Delta Lake tables. Fabric registers them automatically, so they're queryable from Spark, from the lakehouse's **SQL analytics endpoint** (read-only T-SQL), and from Power BI.

The OneLake path of a file looks like:

```
abfss://medvista-data-engineering@onelake.dfs.fabric.microsoft.com/medvista_bronze.Lakehouse/Files/source_drops/epic/encounters_2026-03-17.csv
        └──────── workspace ───────┘                               └── item ──────────────┘ └── path inside the item ─────────────┘
```

### 3.2 Lakehouse schemas and three-part names

New lakehouses have **Lakehouse schemas** turned on by default: tables live in schemas, and `dbo` is the default. In Spark you then name a table `lakehouse.schema.table`:

```python
spark.table("medvista_bronze.dbo.ehr_patient_encounters")
```

Every SK-07 notebook defines its prefixes once — `BRONZE = "medvista_bronze.dbo"` — so if you ever create a lakehouse without schemas, you change one line (`BRONZE = "medvista_bronze"`).

### 3.3 Delta Lake in one minute

A Delta table is a folder of Parquet data files plus a `_delta_log` folder of JSON commit files. The log is what gives you ACID transactions, `MERGE`, time travel (`VERSION AS OF`), and schema enforcement — a write whose columns don't match the table is rejected instead of silently corrupting it.

### 3.4 Why Bronze gets an explicit schema

If you let Spark **infer** a schema, it samples the data and guesses. One `N/A` in a numeric column and the whole column becomes `STRING`; every later `SUM()` then fails or, worse, gets cast and silently drops values. An explicit schema turns Bronze into a **contract**: the Lab_02 load notebook reads every file with this schema and fails loudly on a file that breaks it. You'll see both behaviours in Step 5.

### 3.5 Partitioning by `_extract_date`

Every Bronze table gets an `_extract_date DATE` column and is `PARTITIONED BY (_extract_date)`: each day's extract lands in its own folder, so a re-run of one day touches exactly one partition and later notebooks read one day without scanning the rest. Delta can't partition by an expression such as `DATE(_ingested_at)` — the value has to be a real column, which is why we store it.

---

## 4. Step-by-Step Implementation

### Step 1 — Create the three lakehouses

In `medvista-data-engineering`:

1. **+ New item** → **Lakehouse** → name `medvista_bronze`. Leave **Lakehouse schemas** ticked. **Create**.
2. Repeat for `medvista_silver` and `medvista_gold`.

**Expected output:** each lakehouse opens in the Lakehouse explorer with **Tables** (containing a `dbo` schema) and **Files**. Back in the workspace list, each lakehouse also shows a **SQL analytics endpoint** item with the same name — Fabric creates it automatically.

✅ **Checkpoint:** the workspace lists `medvista_bronze`, `medvista_silver` and `medvista_gold`, each with a SQL analytics endpoint.

### Step 2 — Upload the source drops

1. Open `medvista_bronze` → **Files** → **…** → **New subfolder** → `source_drops`.
2. Open `source_drops` → **…** → **Upload** → **Upload folder** → select `C:\DataEngineering\sk07\data\source_drops\epic` → **Upload**.
3. Repeat for `workday` and `sap`.

**Expected output:** `Files/source_drops/epic`, `.../workday` and `.../sap`, each with three CSVs.

4. Right-click `encounters_2026-03-17.csv` → **Properties** and copy the **ABFS path**. Compare it to the pattern in Section 3.1.

✅ **Checkpoint:** nine files under `Files/source_drops`, and you can point to the workspace, item and path parts of an ABFS path.

> These files stand in for MedVista's source systems (Epic SFTP, Workday API, SAP OData). In Lab_02 a pipeline copies them into a `landing` area exactly as it would copy from the real systems.

### Step 3 — Create your first notebook and attach the lakehouse

1. In the workspace: **+ New item** → **Notebook**. Rename it (top left) to `00_create_bronze_schemas`.
2. In the notebook's **Explorer** pane, add `medvista_bronze` (Add data items → choose the lakehouse). It becomes the **default lakehouse** (pinned): relative paths like `Files/...` and two-part names resolve against it.
3. In the first cell, run:

   ```python
   print(spark.version)
   display(spark.sql("SHOW TABLES IN medvista_bronze.dbo"))
   ```

The first run starts a Spark session (usually a few seconds on the starter pool; up to a couple of minutes if a custom pool has to spin up).

**Expected output:** a Spark version (3.5.x on Fabric Runtime 1.3), and an empty result (no tables yet) — not an error. An empty answer proves Spark can resolve the `medvista_bronze.dbo` namespace.

✅ **Checkpoint:** the cell runs without error. If you get "schema not found", your lakehouse was created without schemas — see Troubleshooting.

### Step 4 — Read a CSV from Files

Add a cell:

```python
df = (spark.read.format("csv")
      .option("header", True)
      .load("Files/source_drops/epic/encounters_2026-03-17.csv"))
print(df.count())
df.printSchema()
display(df.limit(5))
```

**Expected output:** `1070`, and a schema where **every column is `string`** — without a schema or `inferSchema`, CSV gives you text.

✅ **Checkpoint:** 1070 rows; all columns `string`.

### Step 5 — Watch schema inference go wrong, and an explicit schema catch it

Create a tiny file with one bad value and read it both ways:

```python
notebookutils.fs.put("Files/scratch/na_demo.csv", "id,amount\n1,10.50\n2,N/A\n3,7.25\n", True)

inferred = (spark.read.option("header", True).option("inferSchema", True)
            .csv("Files/scratch/na_demo.csv"))
inferred.printSchema()          # amount: string  <- one "N/A" poisoned the column

from pyspark.sql.types import StructType, StructField, StringType, DoubleType
strict = (spark.read.option("header", True).option("mode", "FAILFAST")
          .schema(StructType([StructField("id", StringType()),
                              StructField("amount", DoubleType())]))
          .csv("Files/scratch/na_demo.csv"))
strict.show()                   # raises: malformed record
```

**Expected output:** the inferred schema shows `amount: string`. The strict read raises an error mentioning malformed records (FAILFAST mode).

Clean up:

```python
notebookutils.fs.rm("Files/scratch", True)
```

✅ **Checkpoint:** you've seen inference silently degrade a numeric column and an explicit schema stop the load instead. That is the Bronze contract.

### Step 6 — Create the Bronze tables

Replace the notebook contents with the guided lab's Step 1.4 notebook and run it:

```python
# Fabric Notebook: 00_create_bronze_schemas
# Default lakehouse: medvista_bronze. Run once (safe to re-run: IF NOT EXISTS).

# WHY explicit schema: if Spark infers types from a messy export, one "N/A" in a
# numeric column turns the whole column into STRING and every downstream cast
# silently breaks. Declaring the schema up front makes Bronze a contract: the
# load notebook reads the CSV with this schema and fails loudly if a file breaks it.

# WHY _extract_date: Delta cannot partition by an expression such as
# DATE(_ingested_at), so we store the extract date as a real column and partition
# by it. It is also the column every downstream notebook filters on, which means
# a re-run of any day touches exactly one partition.

BRONZE = "medvista_bronze.dbo"   # lakehouse.schema - use "medvista_bronze" if you unticked "Lakehouse schemas"

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {BRONZE}.ehr_patient_encounters (
        encounter_id        STRING        NOT NULL,
        patient_id          STRING        NOT NULL,
        encounter_type      STRING,
        admission_datetime  TIMESTAMP,
        discharge_datetime  TIMESTAMP,
        department_id       STRING,
        attending_physician STRING,
        primary_diagnosis   STRING,
        secondary_diagnoses ARRAY<STRING>,
        hospital_id         STRING,
        insurance_type      STRING,
        total_charges       DOUBLE,
        _source_system      STRING,
        _ingested_at        TIMESTAMP,
        _file_name          STRING,
        _extract_date       DATE
    )
    USING DELTA
    PARTITIONED BY (_extract_date)
""")

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {BRONZE}.hr_staff_schedules (
        schedule_id     STRING        NOT NULL,
        employee_id     STRING,
        role            STRING,
        department_id   STRING,
        hospital_id     STRING,
        shift_date      DATE,
        shift_start     TIMESTAMP,
        shift_end       TIMESTAMP,
        is_overtime     BOOLEAN,
        _source_system  STRING,
        _ingested_at    TIMESTAMP,
        _file_name      STRING,
        _extract_date   DATE
    )
    USING DELTA
    PARTITIONED BY (_extract_date)
""")

spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {BRONZE}.supply_orders (
        order_id        STRING        NOT NULL,
        item_id         STRING,
        item_name       STRING,
        category        STRING,
        quantity        INT,
        unit_cost       DOUBLE,
        total_cost      DOUBLE,
        department_id   STRING,
        hospital_id     STRING,
        order_date      DATE,
        vendor_id       STRING,
        _source_system  STRING,
        _ingested_at    TIMESTAMP,
        _file_name      STRING,
        _extract_date   DATE
    )
    USING DELTA
    PARTITIONED BY (_extract_date)
""")

display(spark.sql(f"SHOW TABLES IN {BRONZE}"))
print("Bronze schemas created")
```

**Expected output:** a table listing `ehr_patient_encounters`, `hr_staff_schedules` and `supply_orders`, then `Bronze schemas created`.

Inspect what you made:

```python
display(spark.sql("DESCRIBE DETAIL medvista_bronze.dbo.ehr_patient_encounters")
        .select("format", "partitionColumns", "numFiles", "location"))
display(spark.sql("DESCRIBE HISTORY medvista_bronze.dbo.ehr_patient_encounters"))
```

**Expected output:** `format = delta`, `partitionColumns = [_extract_date]`, `numFiles = 0`, a `location` ending in `.../Tables/dbo/ehr_patient_encounters`, and one history row with operation `CREATE TABLE`.

Re-run the creation cell: nothing changes (`IF NOT EXISTS`). Safe re-runs are a habit you'll keep all module.

✅ **Checkpoint:** in the Lakehouse explorer (refresh **Tables**), `dbo` shows the three tables. Switch the top-right selector from **Lakehouse** to **SQL analytics endpoint**: the same three tables are queryable in T-SQL (`SELECT COUNT(*) FROM dbo.ehr_patient_encounters` → `0`).

### Step 7 — Stop the session

Notebook toolbar → **Stop session**. An idle session keeps consuming capacity until it times out.

---

## 5. Production Engineering Practices

1. **Bronze is typed but not corrected.** Types and lineage are added; values are left exactly as the source sent them, so you can always prove what the source said.
2. **Contract-first ingestion.** Declare schemas in code, version them in Git, and fail loudly on violations.
3. **Idempotent DDL.** `CREATE TABLE IF NOT EXISTS` lets the same notebook run in every environment (dev, test, prod) without special cases.
4. **Partition by the column you reprocess by.** Here that's the extract date. Don't partition small tables by high-cardinality columns — thousands of tiny files hurt more than they help.
5. **Keep PHI in as few places as possible.** Uploading to a lakehouse in a restricted workspace is the first access-control decision you make.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `SCHEMA_NOT_FOUND` / `medvista_bronze.dbo` not found | Lakehouse was created without schemas | Use `BRONZE = "medvista_bronze"` in every notebook, or recreate the lakehouse with schemas ticked |
| Spark session takes several minutes to start | Starter pool busy, or a custom pool/environment is attached | Wait; check **Workspace settings → Data Engineering/Science → Spark settings** uses the starter pool |
| `Path does not exist: Files/source_drops/...` | No default lakehouse, or files uploaded to a different folder | Pin `medvista_bronze` in the notebook Explorer; check the folder names in **Files** |
| **Upload folder** not offered | Older browser or blocked folder upload | Create the `epic`, `workday`, `sap` subfolders yourself and use **Upload files** |
| `notebookutils.fs.put` fails on a relative path | No default lakehouse attached | Attach one, or use the full `abfss://...` path from **Properties** |
| New tables don't appear in the explorer | UI not refreshed | **…** next to **Tables** → **Refresh** |
| SQL analytics endpoint doesn't show a new table | Metadata sync runs in the background | Wait a minute and refresh |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"What's the difference between the Tables and Files areas of a lakehouse?"** — Tables holds Delta tables that Fabric registers automatically and exposes to Spark, the SQL analytics endpoint and Power BI. Files holds arbitrary files that only engines you point at them will read.
2. **"Why not let Spark infer the Bronze schema?"** — Inference guesses from a sample; one bad value changes a column's type and the damage surfaces far downstream. An explicit schema makes violations fail at ingestion, where they're cheap to fix.
3. **"Why can't you `PARTITIONED BY (DATE(_ingested_at))`?"** — Delta partitions on stored column values, not expressions. Store the derived value as a real column (`_extract_date`) and partition on that. (Delta has *generated columns* for this in some runtimes; a plain column is the portable answer.)

### Common interview traps

- Calling the SQL analytics endpoint a warehouse. It's read-only T-SQL over lakehouse Delta tables; writes go through Spark (or a Warehouse, Lab_06).
- Assuming three lakehouses means three copies of compute. Storage is separate; compute comes from the shared capacity.

### Key takeaways

- OneLake is one lake; a lakehouse is a folder in it with a managed **Tables** area and an unmanaged **Files** area.
- Tables are named `lakehouse.schema.table`; define the prefix once per notebook.
- Explicit schema + FAILFAST = Bronze contract. `_extract_date` partitions make reprocessing surgical.

---

## Next

[Lab_02 — Pipeline Ingestion: Copy Activities and the Bronze Load](Lab_02_Pipeline_Ingestion_Copy_and_Bronze_Load.md): build the pipeline that lands each day's files and loads them into these tables.
