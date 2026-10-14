# SK-07 END-TO-END GUIDED LAB: MedVista Healthcare Analytics Platform on Microsoft Fabric

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **Microsoft Fabric data engineer** deployed to **MedVista Health**, a regional hospital network with 12 hospitals, 3,000 physicians, and 2 million patient encounters per year. Their data is fragmented across five systems: Epic EHR (patient records), Workday (HR and staffing), SAP (supply chain), a custom bed management system, and lab result feeds.

The Chief Medical Officer cannot answer a simple question like "What is our average ER wait time by day of week and staffing level?" without manually querying three systems. The current state:

- **CMS compliance risk:** Readmission rate reporting takes 2 analysts 3 weeks per quarter. CMS requires accurate submission within 45 days of quarter end — MedVista is consistently late
- **Operational inefficiency:** Bed management is done via whiteboards and phone calls — no real-time visibility into available beds across the 12-hospital network
- **Supply cost overruns:** Supply chain costs increased 23% year-over-year but nobody can correlate supply usage with patient volume to find waste
- **Microsoft ecosystem fit:** MedVista already uses Microsoft 365, Microsoft Entra ID (formerly Azure Active Directory), and Power BI — Fabric is the natural extension with no new vendor relationship

**Target:** A unified Fabric analytics platform with Bronze/Silver/Gold medallion architecture, daily automated pipelines, a Fabric Warehouse for SQL analytics, and a real-time bed utilization dashboard via Eventstreams and KQL.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why three separate Lakehouses (not one)?** | Isolation and security. Raw PHI data in Bronze requires stricter access controls than analytics-ready Gold data. Data quality issues in Bronze (malformed EHR exports) don't propagate to Gold reports. Each layer can be reprocessed independently. |
| **Why Delta MERGE for Silver instead of overwrite?** | Encounter records can be updated hours or days after initial admission (discharge time added, final diagnosis coded). MERGE handles late updates without reloading the full table. Overwrite would lose the previous day's updates. |
| **Why a Fabric Warehouse in addition to the Gold Lakehouse?** | The CMO's analysts know T-SQL but not PySpark. The analytics workspace gets read-only OneLake **shortcuts** to the Gold tables (no copies), and the Warehouse wraps them in T-SQL views. Analytical views in T-SQL are easier to maintain than DAX measures for complex SQL logic. |
| **Why Eventstreams + KQL for bed utilization (not just batch)?** | Bed availability changes every few minutes during peak hours. A daily batch refresh means the dashboard shows bed counts that are 12 hours stale. KQL queries on Eventstream data are current to within seconds. |

---

## WHAT YOU WILL BUILD

```
MedVista Fabric Platform:
│
├── 1. WORKSPACES & LAKEHOUSES
│   ├── medvista-data-engineering (workspace)
│   │   ├── medvista_bronze  (Lakehouse — raw source data + landing files)
│   │   ├── medvista_silver  (Lakehouse — cleaned, conformed)
│   │   ├── medvista_gold    (Lakehouse — star schema)
│   │   ├── Notebooks 00_create_bronze_schemas, 00_land_to_bronze,
│   │   │             01_bronze_to_silver, 02_silver_to_gold
│   │   ├── Pipeline MedVista_Daily_Ingestion
│   │   └── Eventhouse medvista_eventhouse + Eventstream medvista_bed_events
│   └── medvista-analytics (workspace)
│       ├── medvista_gold_serving  (Lakehouse — shortcuts to Gold, no copies)
│       └── medvista_warehouse     (Fabric Warehouse — T-SQL views)
│
├── 2. DAILY INGESTION PIPELINE
│   ├── Set variable: runDate (parameter, or yesterday when empty)
│   ├── Copy ×3 (parallel): source drops → Bronze Files/landing/<system>/dt=<date>/
│   ├── Notebook: landing → Bronze tables (explicit schema, idempotent)
│   ├── Notebook: Bronze → Silver (dedup, conform, MERGE)
│   ├── Notebook: Silver → Gold (star schema, assertions, OPTIMIZE)
│   └── On failure or skip: Teams message, then Fail activity
│
├── 3. SPARK NOTEBOOKS
│   ├── 00_create_bronze_schemas  (run once)
│   ├── 00_land_to_bronze         (CSV → typed Bronze Delta)
│   ├── 01_bronze_to_silver       (dedup, type enforce, readmission flag, MERGE)
│   └── 02_silver_to_gold         (star schema, OPTIMIZE, assertions)
│
├── 4. FABRIC WAREHOUSE VIEWS
│   ├── analytics.vw_hospital_performance
│   ├── analytics.vw_supply_cost_yoy
│   └── analytics.vw_readmission_analysis
│
└── 5. REAL-TIME INTELLIGENCE
    ├── Eventstream: medvista_bed_events (custom endpoint → Eventhouse KQL table)
    └── KQL Queries: current occupancy, admissions trend, turnover time, capacity alert
```

### Time Estimate: 10–12 hours

> **Doing this for the first time?** The [Granular Labs](../Granular%20Labs/) (Lab_00 to Lab_07) walk through every part of this lab one step at a time, with expected output and troubleshooting for each step. Same scenario, same names, same code.

---

## BEFORE YOU START: ACCOUNT, CAPACITY AND COST

| Item | What you need |
|---|---|
| **Account** | A Microsoft Entra **work or school** account. Personal accounts (outlook.com, gmail.com) can't sign up for Fabric. If your organisation has turned off trials, ask your Fabric administrator for a workspace on an existing capacity. |
| **Capacity** | The free **Fabric trial**: 60 days, F4 or F64 size, up to 1 TB of OneLake storage. Every item in this lab works on the trial. Copilot and AI features aren't included in the trial and this lab doesn't use them. |
| **Local machine** | Windows with PowerShell, Python 3.12 (3.11+ is fine) to generate the sample data and run the bed-event simulator. |

**Cost guidance:**

- **On the trial the lab costs $0.** The trial capacity is a fixed size, so the risk is not a bill but **throttling**: if you leave heavy jobs running, everything in the capacity slows down. Stop Spark sessions you aren't using (notebook toolbar → **Stop session**), disable the pipeline schedule after you finish Part 2, and stop the Eventstream after Part 6.
- **On a paid F-SKU (pay-as-you-go)** you pay per hour while the capacity is running, whether or not you use it. An F2 costs roughly US$0.36 per hour (about US$263 per month if never paused) in US regions at the time of writing; check the [Azure pricing calculator](https://azure.microsoft.com/pricing/calculator/?service=microsoft-fabric) for your region. **Pause the capacity in the Azure portal whenever you stop working** — a paused capacity stops compute billing; OneLake storage (a few MB here) is still billed.
- **Real-Time Intelligence is the part that runs continuously.** An active Eventstream and an Eventhouse that is receiving data consume capacity even when nobody is querying. Leave the Eventhouse **Minimum consumption** setting off, and follow the teardown at the end.
- When the trial ends, workspaces drop back to Pro and Fabric items become inaccessible; content is kept for 7 days in case you move the workspace to a paid capacity.

---

## PART 1: WORKSPACE, LAKEHOUSES AND SAMPLE DATA (45 min)

### Step 1.1: Create the Workspaces, Lakehouses and Warehouse

In the Fabric portal ([app.fabric.microsoft.com](https://app.fabric.microsoft.com)), create the following:

```
1. Create workspace: "medvista-data-engineering"
   - Workspace settings → License info / Workspace type: Trial (or your F-SKU capacity)
   - Git integration: connect to Azure DevOps or GitHub (optional but recommended)

2. Create workspace: "medvista-analytics"
   - Same capacity

3. In medvista-data-engineering, create 3 Lakehouses (keep "Lakehouse schemas" ticked — the default):
   - medvista_bronze
   - medvista_silver
   - medvista_gold

4. In medvista-analytics, create (you use these in Part 5):
   - Lakehouse: medvista_gold_serving   (schemas ticked)
   - Warehouse: medvista_warehouse
```

**Naming in code.** Schema-enabled lakehouses put tables in the `dbo` schema, so Spark names are `lakehouse.schema.table`, for example `medvista_bronze.dbo.ehr_patient_encounters`. Every notebook below defines the prefixes once at the top (`BRONZE = "medvista_bronze.dbo"`). If you unticked "Lakehouse schemas", change the prefixes to plain `medvista_bronze` and everything else works unchanged.

### Step 1.2: Generate the Sample Source Data

You don't have access to MedVista's Epic, Workday and SAP systems, so a script creates three days (2026-03-17 to 2026-03-19) of deliberately imperfect extracts that behave like them: duplicate rows, inconsistent encounter-type spellings, negative charges, discharge-before-admission rows, missing insurance, missing supply totals, zero-quantity orders, and encounters that are re-sent a day or two later once the patient is discharged.

Save this as `C:\DataEngineering\sk07\generate_medvista_data.py`:

```python
"""generate_medvista_data.py - MedVista sample source extracts for SK-07.

Creates three days (2026-03-17 .. 2026-03-19) of deliberately imperfect extracts
that simulate the three MedVista source systems:

  source_drops/epic/encounters_YYYY-MM-DD.csv        (stands in for the Epic EHR SFTP export)
  source_drops/workday/schedules_YYYY-MM-DD.csv      (stands in for the Workday REST API)
  source_drops/sap/supply_orders_YYYY-MM-DD.csv      (stands in for the SAP OData feed)

Standard library only. Deterministic: the same seed always produces the same files.
Usage:  python generate_medvista_data.py [output_root]
"""
import csv
import random
import sys
from datetime import datetime, timedelta
from pathlib import Path

SEED = 2026
DATES = ["2026-03-17", "2026-03-18", "2026-03-19"]
ROOT = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(r"C:\DataEngineering\sk07\data")
TS = "%Y-%m-%d %H:%M:%S"

rng = random.Random(SEED)
HOSPITALS = [f"H{i:03d}" for i in range(1, 13)]
HOSP_WEIGHTS = [18, 12, 10, 6, 14, 6, 3, 3, 6, 8, 7, 7]
DEPARTMENTS = ["D-ER", "D-CARD", "D-ORTH", "D-ONC", "D-PED", "D-MED", "D-SURG", "D-ICU"]
ENC_TYPES = {  # raw spelling variants -> base type (Silver standardises them)
    "ER": "E", "ED": "E", "Emergency": "E", "EMERGENCY": "E",
    "IP": "I", "Inpatient": "I", "OP": "O", "outpatient": "O",
    "OBS": "B", "Observation": "B", " Telehealth ": "T",
}
TYPE_WEIGHTS = [10, 8, 6, 4, 9, 7, 12, 8, 4, 3, 2]
STAY_HOURS = {"E": (2, 10), "I": (24, 110), "O": (1, 4), "B": (12, 40), "T": (0.25, 1)}
CHARGES = {"E": (800, 6000), "I": (9000, 60000), "O": (150, 2500), "B": (2500, 12000), "T": (60, 300)}
DIAGNOSES = ["I21.9", "I50.9", "J18.9", "E11.9", "N39.0", "S72.001A", "K35.80", "J44.1",
             "A41.9", "R07.9", "I10", "F32.9", "O80", "M54.5", "C34.90"]
INSURANCE = ["Medicare", "Medicaid", "Commercial", "Self-Pay"]


def ts(dt):
    return dt.strftime(TS)


def write_csv(path, header, rows):
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("w", newline="", encoding="utf-8") as fh:
        w = csv.writer(fh)
        w.writerow(header)
        w.writerows(rows)


# ---------------------------------------------------------------- encounters
ENC_HEADER = ["encounter_id", "patient_id", "encounter_type", "admission_datetime",
              "discharge_datetime", "department_id", "attending_physician",
              "primary_diagnosis", "secondary_diagnoses", "hospital_id",
              "insurance_type", "total_charges"]
still_admitted = []  # encounters exported without discharge, re-sent once discharged


def new_encounter(day, seq, patient_id=None, admit=None):
    raw_type = rng.choices(list(ENC_TYPES), weights=TYPE_WEIGHTS)[0]
    base = ENC_TYPES[raw_type]
    admit = admit or day + timedelta(minutes=rng.randint(0, 23 * 60 + 59))
    lo, hi = STAY_HOURS[base]
    discharge = admit + timedelta(hours=rng.uniform(lo, hi))
    charges = round(rng.uniform(*CHARGES[base]), 2)
    if rng.random() < 0.01:
        charges = -charges                      # data-entry error -> clamped to 0 in Silver
    insurance = rng.choice(INSURANCE) if rng.random() > 0.05 else ""   # ~5% unknown
    secondary = "|".join(rng.sample(DIAGNOSES, rng.randint(0, 3)))
    return {
        "encounter_id": f"E{day:%Y%m%d}{seq:05d}",
        "patient_id": patient_id or f"P{rng.randint(1, 40000):06d}",
        "encounter_type": raw_type,
        "admit": admit, "discharge": discharge,
        "department_id": "D-ER" if base == "E" else rng.choice(DEPARTMENTS),
        "attending_physician": f"DR{rng.randint(1, 600):04d}",
        "primary_diagnosis": rng.choice(DIAGNOSES),
        "secondary_diagnoses": secondary,
        "hospital_id": rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0],
        "insurance_type": insurance, "total_charges": charges,
    }


def enc_row(e, extract_cutoff, swap=False):
    discharge = e["discharge"] if e["discharge"] <= extract_cutoff else None
    admit = e["admit"]
    if swap:                                     # invalid: discharge before admission
        admit, discharge = e["discharge"], e["admit"]
    return [e["encounter_id"], e["patient_id"], e["encounter_type"], ts(admit),
            ts(discharge) if discharge else "", e["department_id"],
            e["attending_physician"], e["primary_diagnosis"], e["secondary_diagnoses"],
            e["hospital_id"], e["insurance_type"], f'{e["total_charges"]:.2f}']


for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    cutoff = day + timedelta(days=1, hours=6)    # extract runs 06:00 next morning
    rows, seq = [], 0
    # 1) late updates: earlier encounters now discharged -> re-sent with discharge time
    for e in list(still_admitted):
        if e["discharge"] <= cutoff:
            rows.append(enc_row(e, cutoff))
            still_admitted.remove(e)
    # 2) today's admissions
    todays = []
    for _ in range(1000):
        seq += 1
        e = new_encounter(day, seq)
        todays.append(e)
    # 3) same-day readmissions (~5%): ER visit, discharged, back again later that day
    for _ in range(50):
        seq += 1
        first = rng.choice([t for t in todays if t["discharge"] < day + timedelta(hours=20)])
        admit = first["discharge"] + timedelta(hours=rng.uniform(0.5, 3))
        todays.append(new_encounter(day, seq, first["patient_id"], admit))
    for e in todays:
        swap = rng.random() < 0.01 and e["discharge"] <= cutoff
        rows.append(enc_row(e, cutoff, swap))
        if e["discharge"] > cutoff:
            still_admitted.append(e)
    rows += rng.sample(rows, 20)                 # exact duplicate rows (re-sent records)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "epic" / f"encounters_{d}.csv", ENC_HEADER, rows)
    print(f"encounters_{d}.csv     rows={len(rows)}")

# ---------------------------------------------------------------- schedules
ROLES = ["Physician", "Hospitalist MD", "Emergency Doctor", "Registered Nurse (RN)",
         "Nurse Practitioner", "LPN", "Charge Nurse", "Patient Care Assistant",
         "Lab Technician", "Radiology Tech", "Nursing Aide", "Pharmacist",
         "Environmental Services", "Unit Clerk", "Medical Intern", "Endoscopy Tech"]
ROLE_WEIGHTS = [8, 4, 4, 22, 5, 6, 4, 10, 6, 5, 8, 4, 8, 6, 3, 2]
SCHED_HEADER = ["schedule_id", "employee_id", "role", "department_id", "hospital_id",
                "shift_date", "shift_start", "shift_end", "is_overtime"]
for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    rows = []
    for i in range(1, 601):
        start = day + timedelta(hours=rng.choice([7, 7, 7, 11, 15, 19, 19, 23]))
        r = rng.random()
        hours = 17 if r < 0.015 else (1.5 if r < 0.025 else rng.choice([8, 8, 10, 12, 12]))
        rows.append([f"S{day:%Y%m%d}{i:04d}", f"EMP{rng.randint(1, 3000):05d}",
                     rng.choices(ROLES, weights=ROLE_WEIGHTS)[0], rng.choice(DEPARTMENTS),
                     rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0], d, ts(start),
                     ts(start + timedelta(hours=hours)),
                     "true" if rng.random() < 0.12 else "false"])
    rows += rng.sample(rows, 6)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "workday" / f"schedules_{d}.csv", SCHED_HEADER, rows)
    print(f"schedules_{d}.csv      rows={len(rows)}")

# ---------------------------------------------------------------- supply orders
CATEGORIES = ["Pharmaceuticals", "Drug - Generic", "Medication Cart", "Surgical Supplies",
              "Orthopedic Implant", "PPE - Gloves", "Mask N95", "Isolation Gown",
              "Lab Reagent", "Rapid Test Kit", "Linen", "Office Supplies"]
ITEMS = [(f"ITM{i:04d}", f"Item {i:04d}", rng.choice(CATEGORIES), round(rng.uniform(0.5, 400), 2))
         for i in range(1, 121)]
SUPPLY_HEADER = ["order_id", "item_id", "item_name", "category", "quantity", "unit_cost",
                 "total_cost", "department_id", "hospital_id", "order_date", "vendor_id"]
for d in DATES:
    day = datetime.strptime(d, "%Y-%m-%d")
    rows = []
    for i in range(1, 401):
        item_id, name, cat, cost = rng.choice(ITEMS)
        qty = 0 if rng.random() < 0.01 else rng.randint(1, 40)
        total = "" if rng.random() < 0.05 else f"{qty * cost:.2f}"    # ~5% missing total
        rows.append([f"PO{day:%Y%m%d}{i:04d}", item_id, name, cat, qty, f"{cost:.2f}", total,
                     rng.choice(DEPARTMENTS), rng.choices(HOSPITALS, weights=HOSP_WEIGHTS)[0],
                     d, f"V{rng.randint(1, 15):02d}"])
    rows += rng.sample(rows, 4)
    rng.shuffle(rows)
    write_csv(ROOT / "source_drops" / "sap" / f"supply_orders_{d}.csv", SUPPLY_HEADER, rows)
    print(f"supply_orders_{d}.csv  rows={len(rows)}")

print(f"Done. Files written under {ROOT / 'source_drops'}")
```

Run it:

```powershell
cd C:\DataEngineering\sk07
python generate_medvista_data.py
```

**Expected output:**

```
encounters_2026-03-17.csv     rows=1070
encounters_2026-03-18.csv     rows=1202
encounters_2026-03-19.csv     rows=1281
schedules_2026-03-17.csv      rows=606
schedules_2026-03-18.csv      rows=606
schedules_2026-03-19.csv      rows=606
supply_orders_2026-03-17.csv  rows=404
supply_orders_2026-03-18.csv  rows=404
supply_orders_2026-03-19.csv  rows=404
Done. Files written under C:\DataEngineering\sk07\data\source_drops
```

The script is seeded, so these counts are the same on every machine.

### Step 1.3: Upload the Source Drops to OneLake

In production the pipeline's Copy activities would read straight from SFTP, the Workday REST API and SAP OData. In this lab, a `source_drops` folder in the Bronze lakehouse stands in for those three systems; only the Copy **source** setting would change in production (see the table in Part 2).

1. Open **medvista_bronze** → **Files** → **…** → **New subfolder** → `source_drops`.
2. Open `source_drops` → **…** → **Upload** → **Upload folder**, and upload the three folders `epic`, `workday` and `sap` from `C:\DataEngineering\sk07\data\source_drops\`.

You should now see `Files/source_drops/epic/encounters_2026-03-17.csv` and its eight siblings.

### Step 1.4: Create Bronze Table Schemas

Create a notebook **00_create_bronze_schemas**. In the notebook's **Explorer** pane, add **medvista_bronze** and make it the default lakehouse (the pinned one). Then run:

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

**Expected:** `SHOW TABLES` lists `ehr_patient_encounters`, `hr_staff_schedules` and `supply_orders` (all empty).

---

## PART 2: FABRIC PIPELINE — DAILY INGESTION (60 min)

### Step 2.1: Landing → Bronze Notebook

The Copy activities land the raw files unchanged; this notebook turns one day's landed files into typed Bronze rows, adding the lineage columns. Create notebook **00_land_to_bronze** with **medvista_bronze** as the default lakehouse. Put `process_date = "2026-03-17"` in the first cell and mark it as the **parameter cell** (cell **…** menu → **Toggle parameter cell**); a pipeline overrides this value at run time.

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

**Why `replaceWhere` and not `append`:** Bronze is still append-only *across* extracts — each day gets its own `_extract_date` partition and earlier partitions are never touched. Re-running the *same* day replaces only that day's partition, so a retried pipeline can't double-load Bronze.

### Step 2.2: Pipeline Parameter, Variable and Activity Configuration

The pipeline is built in the Fabric pipeline designer. The JSON below is a readable summary of each activity's settings for reference and version control — it is not an importable pipeline definition (a Git-integrated workspace stores the real one as `pipeline-content.json`).

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

**Design notes:**

- **Why a variable?** A parameter's default value must be a literal, not an expression. The parameter defaults to an empty string; `Set_Run_Date` turns "empty" into yesterday's date for scheduled runs, and any explicit value (for a backfill) wins.
- **Why Binary copies?** The landing step should move bytes, not interpret them. Parsing and typing happen once, in `00_land_to_bronze`, with an explicit schema.
- **Error handling.** `Notify_Failure` depends on `Transform_Silver_to_Gold` with **Failed** *or* **Skipped**: conditions listed on the *same* upstream activity are OR'd, while dependencies on *different* activities are AND'd. If any earlier activity fails, every activity after it is skipped, so this single dependency catches a failure anywhere. `Fail_Pipeline` runs on **Completed** (after the message succeeds or fails) and makes the run show **Failed** in the Monitor hub; without it, a successful notification would mark the whole run as succeeded.
- **Teams activity.** Choose **Teams** from the activities pane, sign in, and post to a channel or group chat. (Don't use the old `outlook.office.com/webhook` Office 365 connector URLs: Microsoft is retiring them in favour of Teams Workflows.) If your tenant blocks it, use the **Office 365 Outlook** activity instead, or set the activity to **Deactivated** and rely on the Fail activity plus Monitor hub.

**In production, only the Copy sources change:**

| Activity | Lab source | Production source |
|---|---|---|
| Copy_EHR_Encounters | `Files/source_drops/epic` | SFTP connection to the Epic export server |
| Copy_HR_Schedules | `Files/source_drops/workday` | REST connection to Workday, pagination rule `AbsoluteUrl` = `$.next_page_url` |
| Copy_Supply_Orders | `Files/source_drops/sap` | OData connection to SAP `SUPPLY_SRV/Orders` with `$filter` on `OrderDate` |

Build the first part now — `Set_Run_Date`, the three Copy activities and `Load_Landing_to_Bronze` — and test it with `processDate = 2026-03-17`: you should see `Files/landing/epic/dt=2026-03-17/encounters_2026-03-17.csv` (plus the Workday and SAP equivalents) and 1,070 / 606 / 404 Bronze rows for that date. Add the two transform notebooks, `Notify_Failure` and `Fail_Pipeline` once Parts 3 and 4 have created the notebooks, then run the whole pipeline in Step 4.2.

---

## PART 3: BRONZE TO SILVER NOTEBOOK (60 min)

### Step 3.1: Complete Transformation Notebook

Create notebook **01_bronze_to_silver**, set **medvista_silver** as the default lakehouse and add **medvista_bronze** as a second data item. Mark the first cell as the parameter cell.

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

**Expected row counts** (run the days in order — the readmission look-back and the late-discharge MERGE depend on earlier days being present):

| process_date | Encounters raw / clean / rejected | MERGE result | Schedules written | Supply written |
|---|---|---|---|---|
| 2026-03-17 | 1,070 / 1,041 / 29 | table created, 1,041 rows | 600 | 396 |
| 2026-03-18 | 1,202 / 1,173 / 29 | 1,041 inserted, 132 updated (late discharges) | 600 | 392 |
| 2026-03-19 | 1,281 / 1,248 / 33 | 1,037 inserted, 211 updated | 600 | 394 |

`rejected` = exact duplicates (20 per day) + rows whose discharge is before admission. After three days `silver_encounters` holds **3,119** rows.

---

## PART 4: SILVER TO GOLD NOTEBOOK (60 min)

### Step 4.1: Star Schema Construction with OPTIMIZE and Assertions

Create notebook **02_silver_to_gold**, set **medvista_gold** as the default lakehouse and add **medvista_silver** as a second data item. Mark the first cell as the parameter cell.

```python
# Fabric Notebook: 02_silver_to_gold
# Default lakehouse: medvista_gold. medvista_silver is also added in the Explorer.
# Builds the Gold star schema, runs post-load assertions, then compacts the facts.

# ── Parameters (this cell is marked as the PARAMETER cell) ──────────────────
process_date = "2026-03-17"

# ── Code ─────────────────────────────────────────────────────────────────────
import json
from pyspark.sql import functions as F
from pyspark.sql.utils import AnalysisException
from delta.tables import DeltaTable

SILVER = "medvista_silver.dbo"
GOLD = "medvista_gold.dbo"
date_key_int = int(process_date.replace("-", ""))
print(f"Silver -> Gold for: {process_date}")


def table_exists(name):
    try:
        spark.sql(f"DESCRIBE TABLE {name}")
        return True
    except AnalysisException:
        return False


# ════════════════════════════════════════════════════════════════════
# DIMENSION: dim_date (built once, 2024-2027)
# ════════════════════════════════════════════════════════════════════

if not table_exists(f"{GOLD}.dim_date"):
    dim_date = (
        spark.sql("SELECT explode(sequence(to_date('2024-01-01'), to_date('2027-12-31'), "
                  "interval 1 day)) AS full_date")
        .select(
            F.date_format("full_date", "yyyyMMdd").cast("int").alias("date_key"),
            "full_date",
            F.year("full_date").alias("year_number"),
            F.concat(F.year("full_date").cast("string"), F.lit("-Q"),
                     F.quarter("full_date").cast("string")).alias("quarter_name"),
            F.month("full_date").alias("month_number"),
            F.date_format("full_date", "MMMM").alias("month_name"),
            F.date_format("full_date", "EEEE").alias("day_name"),
            F.dayofweek("full_date").isin(1, 7).alias("is_weekend"),
        )
    )
    dim_date.write.format("delta").mode("overwrite").saveAsTable(f"{GOLD}.dim_date")
    print("dim_date created")


# ════════════════════════════════════════════════════════════════════
# DIMENSION: dim_hospital (static reference data - full reload)
# ════════════════════════════════════════════════════════════════════

dim_hospital = spark.createDataFrame([
    ("H001", "MedVista Downtown",    "Metro City", "East",  450, "Level I Trauma"),
    ("H002", "MedVista Northside",   "Northville", "North", 320, "Community"),
    ("H003", "MedVista Eastside",    "Eastport",   "East",  280, "Community"),
    ("H004", "MedVista Southgate",   "Southdale",  "South", 200, "Critical Access"),
    ("H005", "MedVista West Campus", "Westfield",  "West",  380, "Teaching"),
    ("H006", "MedVista Children's",  "Metro City", "East",  150, "Pediatric Specialty"),
    ("H007", "MedVista Rehab",       "Greenwood",  "North", 100, "Rehabilitation"),
    ("H008", "MedVista Behavioral",  "Lakeside",   "West",   80, "Psychiatric"),
    ("H009", "MedVista Heart Ctr",   "Metro City", "East",  120, "Cardiac Specialty"),
    ("H010", "MedVista Surgical",    "Riverside",  "South", 160, "Ambulatory Surgical"),
    ("H011", "MedVista Urgent N",    "Northville", "North",  30, "Urgent Care"),
    ("H012", "MedVista Urgent S",    "Southdale",  "South",  30, "Urgent Care"),
], schema="hospital_id STRING, hospital_name STRING, city STRING, region STRING, "
          "bed_count INT, hospital_type STRING")

# Stable surrogate key derived from the business key: H007 -> 7 on every run.
# (monotonically_increasing_id() is NOT stable across runs or partitions - a full
#  reload could renumber hospitals and silently re-point every fact row.)
dim_hospital = dim_hospital.withColumn(
    "hospital_key", F.expr("CAST(substring(hospital_id, 2) AS INT)"))

(dim_hospital.write.format("delta").mode("overwrite")
 .saveAsTable(f"{GOLD}.dim_hospital"))


# ════════════════════════════════════════════════════════════════════
# FACT: fact_encounters (incremental MERGE on encounter_id)
# ════════════════════════════════════════════════════════════════════

silver_enc = (
    spark.table(f"{SILVER}.silver_encounters")
    .filter(F.col("_process_date") == process_date)   # new rows AND rows updated today
)

fact_enc = (
    silver_enc
    .join(dim_hospital.select("hospital_id", "hospital_key"), on="hospital_id", how="left")
    .withColumn("admission_date_key",
        F.date_format("admission_datetime", "yyyyMMdd").cast("int"))
    .withColumn("discharge_date_key",
        F.when(F.col("discharge_datetime").isNotNull(),
               F.date_format("discharge_datetime", "yyyyMMdd").cast("int")))
    .select(
        "encounter_id", "patient_id",
        "hospital_key", "department_id",
        "admission_date_key", "discharge_date_key",
        "attending_physician", "encounter_type",
        "primary_diagnosis", "insurance_type",
        "length_of_stay_hours", "total_charges",
        "is_readmission",
        F.current_timestamp().alias("etl_loaded_at"),
    )
)

if table_exists(f"{GOLD}.fact_encounters"):
    (DeltaTable.forName(spark, f"{GOLD}.fact_encounters").alias("t")
     .merge(fact_enc.alias("s"), "t.encounter_id = s.encounter_id")
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())
else:
    fact_enc.write.format("delta").mode("overwrite").saveAsTable(f"{GOLD}.fact_encounters")

enc_count = fact_enc.count()
print(f"fact_encounters: {enc_count:,} records for {process_date}")


# ════════════════════════════════════════════════════════════════════
# FACT: fact_supply_costs (partition-scoped overwrite on date_key)
# ════════════════════════════════════════════════════════════════════

fact_supply = (
    spark.table(f"{SILVER}.silver_supply_orders")
    .filter(F.col("order_date") == process_date)
    .join(dim_hospital.select("hospital_id", "hospital_key"), on="hospital_id", how="left")
    .groupBy("hospital_key", "department_id", "category_group",
             F.date_format("order_date", "yyyyMMdd").cast("int").alias("date_key"))
    .agg(
        F.count("order_id").alias("order_count"),
        F.sum("quantity").alias("total_quantity"),
        F.sum("total_cost").alias("total_cost"),
        F.avg("unit_cost").alias("avg_unit_cost"),
        F.countDistinct("item_id").alias("unique_items"),
        F.countDistinct("vendor_id").alias("unique_vendors"),
    )
    .withColumn("etl_loaded_at", F.current_timestamp())
)

writer = fact_supply.write.format("delta").mode("overwrite")
if table_exists(f"{GOLD}.fact_supply_costs"):
    writer = writer.option("replaceWhere", f"date_key = {date_key_int}")
writer.saveAsTable(f"{GOLD}.fact_supply_costs")
supply_rows = fact_supply.count()


# ════════════════════════════════════════════════════════════════════
# POST-LOAD QUALITY ASSERTIONS
# A failed assert raises an exception -> the notebook fails -> the pipeline fails.
# ════════════════════════════════════════════════════════════════════

# 1: Encounter count is plausible for one day
assert 100 <= enc_count <= 50_000, \
    f"Encounter count {enc_count} outside expected range [100, 50000]"

# 2: Every encounter found its hospital (a null key drops rows from every hospital report)
null_hospital_keys = fact_enc.filter(F.col("hospital_key").isNull()).count()
assert null_hospital_keys == 0, \
    f"{null_hospital_keys} encounter rows have null hospital_key - missing dim_hospital entry?"

# 3: No negative charges reached Gold
neg_charges = fact_enc.filter(F.col("total_charges") < 0).count()
assert neg_charges == 0, f"{neg_charges} encounter rows have negative total_charges"

# 4: Readmission rate is plausible (< 30%) across the whole fact table
gold_fact = spark.table(f"{GOLD}.fact_encounters")
total_enc = gold_fact.count()
readmit_count = gold_fact.filter(F.col("is_readmission")).count()
readmit_rate = readmit_count / total_enc if total_enc else 0.0
assert readmit_rate < 0.30, \
    f"Readmission rate {readmit_rate:.1%} exceeds 30% - data quality issue?"

print(f"All assertions passed (readmission rate {readmit_rate:.1%})")


# ════════════════════════════════════════════════════════════════════
# OPTIMIZE AND ZORDER
# MERGE and small daily writes leave many small files; compact them and co-locate
# rows by the columns most queries filter on.
# ════════════════════════════════════════════════════════════════════

spark.sql(f"OPTIMIZE {GOLD}.fact_encounters ZORDER BY (hospital_key, admission_date_key)")
# WHY these columns: most CMO dashboard queries filter by hospital and date range.
spark.sql(f"OPTIMIZE {GOLD}.fact_supply_costs ZORDER BY (hospital_key, date_key)")

print(f"\nSilver -> Gold complete for {process_date}")
notebookutils.notebook.exit(json.dumps({
    "process_date": process_date,
    "fact_encounters_written": enc_count,
    "fact_supply_rows": supply_rows,
    "readmission_rate": round(readmit_rate, 4),
    "assertions": "passed",
}))
```

### Step 4.2: Run the Pipeline for Three Days

Run **MedVista_Daily_Ingestion** three times, waiting for each run to finish, with `processDate` = `2026-03-17`, then `2026-03-18`, then `2026-03-19`.

**Expected after the third run:**

| Check | Value |
|---|---|
| `fact_encounters` rows | 3,119 |
| Rows written per day (`fact_encounters_written` exit value) | 1,041 / 1,173 / 1,248 |
| Readmission rate after each day | 5.8% / 6.8% / 7.9% |
| `fact_supply_costs` rows per day | 255 / 251 / 249 |
| `dim_hospital` / `dim_date` rows | 12 / 1,461 |

The per-day figures assume each date goes through Silver *and* Gold before the next date starts, which is what the pipeline does. If you ran `01_bronze_to_silver` by hand for all three dates first, each Gold run only picks up the rows whose latest change happened on that date (844 / 1,027 / 1,248), but the final totals are identical. To reproduce the table exactly, drop the Silver and Gold tables and run the pipeline again — Bronze is the replayable source of truth.

Then:

1. **Re-run 2026-03-19.** Row counts must not change — every write is a MERGE or a partition-scoped overwrite.
2. **Prove the alerting path.** Run with `processDate = 2026-03-20` (no files exist). The three Copy activities fail, everything downstream is skipped, `Notify_Failure` posts, and the run ends **Failed** with `MEDVISTA_LOAD_FAILED`.
3. **Schedule it.** Pipeline → **Schedule** → daily at 06:30 in your time zone, leaving `processDate` empty. **Turn the schedule off** when you finish the lab — the sample data has no files for "yesterday", so scheduled runs will fail by design.

---

## PART 5: FABRIC WAREHOUSE VIEWS (45 min)

### Step 5.1: Expose Gold to the Analytics Workspace with Shortcuts

A Warehouse can query other warehouses and lakehouse SQL analytics endpoints **in the same workspace** using three-part names. The Gold lakehouse lives in a different workspace, so bring its tables over as **OneLake shortcuts** — pointers, not copies:

1. Open **medvista_gold_serving** (in `medvista-analytics`).
2. **Tables** → **dbo** → **…** → **New table shortcut** → **Microsoft OneLake** → select **medvista_gold** → tick `fact_encounters`, `fact_supply_costs`, `dim_hospital`, `dim_date` → **Create**.
3. Open the SQL analytics endpoint of `medvista_gold_serving` and run `SELECT COUNT(*) FROM dbo.fact_encounters;` — expected **3,119**. (If a new table doesn't appear yet, wait a minute: the SQL analytics endpoint syncs Delta metadata in the background.)

Analysts get roles in `medvista-analytics` only. OneLake also checks permissions on a shortcut's target, so test the views with a real analyst account before go-live and grant the minimum your organisation's OneLake security model requires on `medvista_gold`.

### Step 5.2: Create the Analytical Views

Open **medvista_warehouse** → **New SQL query**. In the Explorer, select **+ Warehouses** and add `medvista_gold_serving` so its tables can be referenced as `medvista_gold_serving.dbo.<table>`. Run each statement on its own (highlight it, then **Run**):

```sql
-- ════════════════════════════════════════════════════════════════════
-- STEP 0: Schema for analyst-facing objects
-- ════════════════════════════════════════════════════════════════════

IF SCHEMA_ID('analytics') IS NULL
    EXEC('CREATE SCHEMA analytics');


-- ════════════════════════════════════════════════════════════════════
-- VIEW 1: Hospital Performance Summary (daily grain)
-- Used by: CMO monthly dashboard, CMS compliance reports
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_hospital_performance AS
SELECT
    h.hospital_name,
    h.region,
    h.hospital_type,
    h.bed_count,
    d.full_date        AS report_date,
    d.month_name,
    d.year_number,
    d.quarter_name,

    COUNT(DISTINCT f.encounter_id)                          AS encounter_count,
    COUNT(DISTINCT f.patient_id)                            AS unique_patients,
    AVG(f.length_of_stay_hours)                             AS avg_los_hours,
    MEDIAN(f.length_of_stay_hours)                          AS median_los_hours,

    -- CMS Readmission metric
    SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END)  AS readmission_count,
    CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
        / NULLIF(COUNT(DISTINCT f.encounter_id), 0)         AS readmission_rate,

    -- CMS status (target < 15.5% for most conditions)
    CASE
        WHEN CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
             / NULLIF(COUNT(DISTINCT f.encounter_id), 0) > 0.155
        THEN 'ABOVE TARGET' ELSE 'WITHIN TARGET'
    END                                                     AS cms_readmission_status,

    SUM(f.total_charges)                                    AS total_charges,
    AVG(f.total_charges)                                    AS avg_charge_per_encounter

FROM medvista_gold_serving.dbo.fact_encounters   f
JOIN medvista_gold_serving.dbo.dim_hospital      h ON f.hospital_key = h.hospital_key
JOIN medvista_gold_serving.dbo.dim_date          d ON f.admission_date_key = d.date_key
GROUP BY
    h.hospital_name, h.region, h.hospital_type, h.bed_count,
    d.full_date, d.month_name, d.year_number, d.quarter_name;


-- ════════════════════════════════════════════════════════════════════
-- VIEW 2: Supply Cost Year-over-Year Analysis
-- Used by: CFO supply cost dashboard, contract negotiation prep
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_supply_cost_yoy AS
WITH monthly AS (
    SELECT
        h.hospital_name, h.region, s.category_group,
        d.year_number, d.month_number, d.month_name,
        SUM(s.total_cost)     AS monthly_cost,
        SUM(s.total_quantity) AS monthly_quantity
    FROM medvista_gold_serving.dbo.fact_supply_costs s
    JOIN medvista_gold_serving.dbo.dim_hospital      h ON s.hospital_key = h.hospital_key
    JOIN medvista_gold_serving.dbo.dim_date          d ON s.date_key = d.date_key
    GROUP BY h.hospital_name, h.region, s.category_group,
             d.year_number, d.month_number, d.month_name
),
with_lag AS (
    SELECT
        hospital_name, region, category_group,
        year_number, month_number, month_name,
        monthly_cost, monthly_quantity,
        LAG(monthly_cost) OVER (
            PARTITION BY hospital_name, category_group, month_number
            ORDER BY year_number
        ) AS prior_year_cost
    FROM monthly
)
SELECT
    hospital_name, region, category_group,
    year_number, month_number, month_name,
    monthly_cost, monthly_quantity, prior_year_cost,
    CASE
        WHEN prior_year_cost IS NOT NULL AND prior_year_cost > 0
        THEN ROUND((monthly_cost - prior_year_cost) / prior_year_cost * 100, 2)
        ELSE NULL
    END AS yoy_cost_change_pct
FROM with_lag;


-- ════════════════════════════════════════════════════════════════════
-- VIEW 3: Readmission Analysis (monthly, by hospital and diagnosis)
-- Used by: CMS quarterly submission, quality-improvement teams
-- ════════════════════════════════════════════════════════════════════

CREATE OR ALTER VIEW analytics.vw_readmission_analysis AS
SELECT
    h.hospital_name,
    h.region,
    d.year_number,
    d.quarter_name,
    d.month_number,
    d.month_name,
    f.encounter_type,
    f.primary_diagnosis,
    COUNT(*)                                                AS encounter_count,
    SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END)  AS readmission_count,
    CAST(SUM(CASE WHEN f.is_readmission = 1 THEN 1 ELSE 0 END) AS FLOAT)
        / NULLIF(COUNT(*), 0)                               AS readmission_rate,
    AVG(f.length_of_stay_hours)                             AS avg_los_hours
FROM medvista_gold_serving.dbo.fact_encounters   f
JOIN medvista_gold_serving.dbo.dim_hospital      h ON f.hospital_key = h.hospital_key
JOIN medvista_gold_serving.dbo.dim_date          d ON f.admission_date_key = d.date_key
GROUP BY
    h.hospital_name, h.region, d.year_number, d.quarter_name,
    d.month_number, d.month_name, f.encounter_type, f.primary_diagnosis;
```

**Verify:**

```sql
SELECT COUNT(*) AS daily_rows FROM analytics.vw_hospital_performance;   -- 36 (12 hospitals × 3 days)

SELECT SUM(readmission_count) AS readmissions, SUM(encounter_count) AS encounters
FROM analytics.vw_readmission_analysis;                                 -- 246 / 3,119

SELECT hospital_name, report_date, readmission_rate
FROM analytics.vw_hospital_performance
WHERE cms_readmission_status = 'ABOVE TARGET';                          -- 1 row: MedVista Urgent S,
                                                                        -- 2026-03-19, 0.1607 (9 of 56)

SELECT TOP 5 * FROM analytics.vw_supply_cost_yoy;                       -- prior_year_cost is NULL:
                                                                        -- there is no March 2025 data yet
```

`MEDIAN()` is an aggregate available in Fabric Warehouse and SQL analytics endpoints. In SQL Server, `PERCENTILE_CONT` is analytic-only (it needs an `OVER` clause), so the familiar `PERCENTILE_CONT(0.5) WITHIN GROUP (...)` plus `GROUP BY` pattern is not portable; `MEDIAN()` states the intent directly.

---

## PART 6: REAL-TIME INTELLIGENCE (60 min)

### Step 6.1: Eventhouse and KQL Table

1. In `medvista-data-engineering`, create an **Eventhouse** named `medvista_eventhouse`. Inside it, add a KQL database named `medvista_realtime` (**+ Database**). Leave **Minimum consumption** off.
2. Open `medvista_realtime`, open its query editor (the KQL queryset that comes with the database), and run this management command once, so the table has explicit types instead of inferred ones:

```kql
.create table bed_events (
    event_id: string,
    bed_id: string,
    ward_id: string,
    hospital_id: string,
    event_type: string,
    event_timestamp: datetime,
    patient_id: string
)
```

### Step 6.2: Eventstream Configuration

In the Fabric portal:

1. Create Eventstream **medvista_bed_events**.
2. Source: **Use custom endpoint** (name it `bed_management_system`). This is an Event Hubs-compatible endpoint — in production the bed management system, or an Azure Event Hub it already publishes to, connects here.
3. Destination 1: **Eventhouse** → ingestion mode **Event processing before ingestion** → database `medvista_realtime` → existing table `bed_events` → input format **JSON**.
4. Destination 2 (optional, long-term storage): **Lakehouse** `medvista_gold`, new table `bed_events_archive`.
5. **Publish**. Then select the custom endpoint source → **Details** → **Event Hub** → **SAS Key Authentication**, and copy **Connection string-primary key**.

### Step 6.3: Stream Bed Events

The simulator stands in for the bed management system: 240 beds (12 hospitals × 20 beds) emitting ADMIT / DISCHARGE / CLEAN / MAINTENANCE events. It first replays 24 hours of history, then streams one live step per second. Save it as `C:\DataEngineering\sk07\bed_event_simulator.py`:

```python
"""bed_event_simulator.py - stands in for MedVista's bed management system.

Sends ADMIT / DISCHARGE / CLEAN / MAINTENANCE events for 240 beds (12 hospitals x
20 beds) to a Fabric Eventstream custom endpoint, using the Event Hubs protocol.

  1. Backfill: replays the last N hours (default 24) with past timestamps, so the
     KQL queries have history to work with immediately.
  2. Live: one event every ~1 second for --minutes minutes (default 15).

Usage (PowerShell):
  python bed_event_simulator.py --dry-run --minutes 1                  # print only, no Azure needed
  $env:MEDVISTA_ES_CONN = "<Connection string-primary key>"
  python bed_event_simulator.py --minutes 15

--dry-run uses the standard library only. Sending needs:  python -m pip install azure-eventhub
"""
import argparse
import json
import os
import random
import sys
import time
import uuid
from datetime import datetime, timedelta, timezone

# Target occupancy per hospital: H001, H005 and H009 run hot so the capacity alert fires.
TARGET_OCCUPANCY = {"H001": 0.97, "H002": 0.80, "H003": 0.75, "H004": 0.60,
                    "H005": 0.92, "H006": 0.70, "H007": 0.65, "H008": 0.78,
                    "H009": 0.87, "H010": 0.55, "H011": 0.40, "H012": 0.45}
WARDS_PER_HOSPITAL, BEDS_PER_WARD = 4, 5


def build_beds(rng):
    beds = {}
    for hospital, target in TARGET_OCCUPANCY.items():
        for w in range(1, WARDS_PER_HOSPITAL + 1):
            for b in range(1, BEDS_PER_WARD + 1):
                bed_id = f"{hospital}-W{w}-B{b:02d}"
                state = "ADMIT" if rng.random() < target else "CLEAN"
                beds[bed_id] = {"hospital_id": hospital, "ward_id": f"{hospital}-W{w}",
                                "state": state, "target": target}
    return beds


def next_state(bed, rng):
    """Return the next event type for this bed, or None if nothing happens."""
    s, target = bed["state"], bed["target"]
    if s == "ADMIT":
        # Discharge probability chosen so long-run occupancy is close to the target
        return "DISCHARGE" if rng.random() < (1 - target) / (1 + target) else None
    if s == "DISCHARGE":
        return "CLEAN"                                  # housekeeping always finishes
    if s == "MAINTENANCE":
        return "CLEAN" if rng.random() < 0.5 else None
    # CLEAN (available)
    if rng.random() < 0.02:
        return "MAINTENANCE"
    return "ADMIT" if rng.random() < target else None


def make_event(bed_id, bed, event_type, ts, rng):
    return {
        "event_id": str(uuid.UUID(int=rng.getrandbits(128))),
        "bed_id": bed_id,
        "ward_id": bed["ward_id"],
        "hospital_id": bed["hospital_id"],
        "event_type": event_type,
        "event_timestamp": ts.strftime("%Y-%m-%dT%H:%M:%SZ"),
        "patient_id": f"P{rng.randint(1, 40000):06d}" if event_type == "ADMIT" else "",
    }


def step(beds, ts, rng):
    bed_id = rng.choice(sorted(beds))
    bed = beds[bed_id]
    event_type = next_state(bed, rng)
    if event_type is None:
        return None
    bed["state"] = event_type
    return make_event(bed_id, bed, event_type, ts, rng)


class Sender:
    def __init__(self, conn_str, dry_run):
        self.dry_run, self.sent = dry_run, 0
        if not dry_run:
            try:
                from azure.eventhub import EventHubProducerClient, EventData
            except ImportError:
                sys.exit("azure-eventhub is not installed: python -m pip install azure-eventhub")
            self.EventData = EventData
            # The custom endpoint connection string already contains EntityPath=es_...
            self.client = EventHubProducerClient.from_connection_string(conn_str)

    def send(self, events):
        if not events:
            return
        if self.dry_run:
            for e in events:
                print(json.dumps(e))
        else:
            batch = self.client.create_batch()
            for e in events:
                try:
                    batch.add(self.EventData(json.dumps(e)))
                except ValueError:                      # batch full -> send and start a new one
                    self.client.send_batch(batch)
                    batch = self.client.create_batch()
                    batch.add(self.EventData(json.dumps(e)))
            self.client.send_batch(batch)
        self.sent += len(events)

    def close(self):
        if not self.dry_run:
            self.client.close()


def main():
    p = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    p.add_argument("--connection-string", default=os.environ.get("MEDVISTA_ES_CONN"))
    p.add_argument("--backfill-hours", type=float, default=24)
    p.add_argument("--minutes", type=float, default=15)
    p.add_argument("--seed", type=int, default=7)
    p.add_argument("--dry-run", action="store_true", help="print events instead of sending")
    args = p.parse_args()
    if not args.dry_run and not args.connection_string:
        sys.exit("Set MEDVISTA_ES_CONN or pass --connection-string (or use --dry-run)")

    rng = random.Random(args.seed)
    beds = build_beds(rng)
    sender = Sender(args.connection_string, args.dry_run)
    now = datetime.now(timezone.utc).replace(microsecond=0)

    # 1) Backfill: initial state of every bed, then ~one step per 15 simulated seconds
    t = now - timedelta(hours=args.backfill_hours)
    sender.send([make_event(b, beds[b], beds[b]["state"], t, rng) for b in sorted(beds)])
    batch = []
    while t < now:
        t += timedelta(seconds=15)
        e = step(beds, t, rng)
        if e:
            batch.append(e)
        if len(batch) >= 200:
            sender.send(batch)
            batch = []
    sender.send(batch)
    print(f"Backfill complete: {sender.sent} events covering {args.backfill_hours:g}h", file=sys.stderr)

    # 2) Live: one step per second, real timestamps
    try:
        for _ in range(int(args.minutes * 60)):
            e = step(beds, datetime.now(timezone.utc).replace(microsecond=0), rng)
            if e:
                sender.send([e])
            if not args.dry_run:
                time.sleep(1)
    except KeyboardInterrupt:
        print("Stopped by user", file=sys.stderr)
    finally:
        sender.close()
    occupied = sum(1 for b in beds.values() if b["state"] == "ADMIT")
    print(f"Done. {sender.sent} events sent. Occupied beds now: {occupied}/{len(beds)}", file=sys.stderr)


if __name__ == "__main__":
    main()
```

```powershell
cd C:\DataEngineering\sk07
python bed_event_simulator.py --dry-run --minutes 1        # check the event shape, sends nothing
python -m pip install azure-eventhub
$env:MEDVISTA_ES_CONN = "<paste Connection string-primary key>"
python bed_event_simulator.py --minutes 15
```

**Expected:** `Backfill complete: 2149 events covering 24h`, then the live stream (the count is the same every time because the simulator is seeded).  Within a minute the Eventstream canvas shows events flowing, and `bed_events | count` in the KQL database returns a few thousand rows.

### Step 6.4: KQL Queries for Real-Time Dashboards

```kql
// ── Query 1: Current bed occupancy across all hospitals ──────────────────────
// arg_max returns the most recent event_type per bed — its current status.
// The look-back must be longer than the longest gap between two events for one
// bed: a patient can occupy a bed for days without a new event.
bed_events
| where event_timestamp > ago(7d)
| summarize arg_max(event_timestamp, event_type) by bed_id, ward_id, hospital_id
| summarize
    total_beds    = count(),
    occupied      = countif(event_type == "ADMIT"),
    available     = countif(event_type == "CLEAN"),
    being_cleaned = countif(event_type == "DISCHARGE"),
    maintenance   = countif(event_type == "MAINTENANCE")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total_beds) * 100, 1)
| order by occupancy_pct desc
| render barchart with (title="Current Bed Occupancy by Hospital")


// ── Query 2: Hourly admissions trend (24 hours) ──────────────────────────────
bed_events
| where event_timestamp > ago(24h) and event_type == "ADMIT"
| summarize admits = count() by hospital_id, hour = bin(event_timestamp, 1h)
| render timechart with (title="Admissions per Hour")


// ── Query 3: Bed turnover time (discharge → next admit on the SAME bed) ──────
// WHY: Faster turnaround = more effective capacity without adding beds.
// next() reads the following row in sort order; the bed_id check stops a
// DISCHARGE on one bed being paired with an ADMIT on the next bed in the list.
bed_events
| where event_timestamp > ago(7d)
| where event_type in ("DISCHARGE", "ADMIT")
| sort by bed_id asc, event_timestamp asc
| extend next_bed = next(bed_id), next_event = next(event_type), next_time = next(event_timestamp)
| where event_type == "DISCHARGE" and next_event == "ADMIT" and next_bed == bed_id
| extend turnover_minutes = datetime_diff('minute', next_time, event_timestamp)
| summarize
    avg_turnover_min = round(avg(turnover_minutes), 0),
    p90_turnover_min = round(percentile(turnover_minutes, 90), 0)
    by hospital_id, day = bin(event_timestamp, 1d)
| project day, hospital_id, avg_turnover_min
| render timechart with (title="Daily Average Bed Turnover Time (Minutes)")


// ── Query 4: Alert — hospitals approaching capacity (>= 85%) ─────────────────
// Same 7-day state look-back as Query 1: a 1-hour window would only see beds
// that happened to change in the last hour and report nonsense occupancy.
bed_events
| where event_timestamp > ago(7d)
| summarize arg_max(event_timestamp, event_type) by bed_id, hospital_id
| summarize
    total    = count(),
    occupied = countif(event_type == "ADMIT")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total) * 100, 1)
| where occupancy_pct >= 85
| project hospital_id, occupancy_pct, occupied, total,
    beds_not_occupied = total - occupied,
    alert_level = case(
        occupancy_pct >= 95, "CRITICAL",
        occupancy_pct >= 90, "WARNING",
        "WATCH"
    )
| order by occupancy_pct desc
```

**Expected:** Query 1 shows 12 hospitals with 20 beds each. Query 4 returns the hot hospitals — typically H001 (CRITICAL), H005 (WARNING) and H009 (WATCH); exact percentages drift while the simulator runs. Pin Query 1 and Query 4 to a **Real-Time Dashboard** (**Pin to dashboard** in the queryset) for the bed-management team.

---

## TEARDOWN AND COST CONTROL

When you finish (or stop for the day):

1. Stop the simulator (Ctrl+C) and, in the Eventstream, **Deactivate** the source and destinations (or delete the Eventstream).
2. Turn off the pipeline schedule.
3. Stop any running notebook sessions (**Stop session**).
4. Paid capacity only: **Pause** the capacity in the Azure portal.
5. Keep the workspaces if you are continuing to the capstone; otherwise delete them (workspace settings → **Remove this workspace**).

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The Bronze-to-Silver notebook uses `dropDuplicates(["encounter_id"])` before the Delta MERGE. If instead you ran the MERGE without deduplication, and today's Bronze extract contained the same encounter_id twice with different discharge times, what would happen in the MERGE? Why does Delta MERGE behave this way, and how would you fix it so the *latest* version wins?

2. The Silver-to-Gold notebook uses `OPTIMIZE ... ZORDER BY (hospital_key, admission_date_key)`. After a week of daily pipelines, you notice that queries filtering on `encounter_type` (e.g., "show me all EMERGENCY encounters") are slow. Would adding `encounter_type` to the ZORDER clause help? What is the practical limit to how many columns you should include in ZORDER, and why?

3. The occupancy queries look back 7 days before `arg_max`. A bed in Hospital H002 has had no events for 8 days (a long-stay patient with no status update). What does the current occupancy query return for that bed, and what change to the design — in the source system or in KQL — would make current-state queries correct no matter how long a bed stays unchanged?

4. The pipeline has three parallel Copy activities (EHR, HR Schedules, Supply Orders) that all complete before the Bronze load starts. In production the EHR Copy takes 45 minutes, HR takes 5 minutes, and Supply takes 8 minutes. If you need to reduce the total pipeline duration, which activity should you focus on, and what are two Fabric-native approaches to reducing its run time? (Also consider the three notebook activities: what does each one spend on Spark session start-up, and how could they share a session?)

5. The CMO asks you to add patient age and gender to `fact_encounters` for demographic analysis. Currently, age and gender are in the Epic EHR source but are not ingested. Walk through the exact changes needed at each layer: Bronze table schema and load-notebook schema, Silver transformation addition, Gold MERGE update, Warehouse view update, and Power BI semantic model update. Which step has the highest risk of breaking existing downstream artefacts?
