# SK-07 END-TO-END GUIDED LAB: MedVista Healthcare Analytics Platform on Microsoft Fabric

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **Microsoft Fabric data engineer** deployed to **MedVista Health**, a regional hospital network with 12 hospitals, 3,000 physicians, and 2 million patient encounters per year. Their data is fragmented across five systems: Epic EHR (patient records), Workday (HR and staffing), SAP (supply chain), a custom bed management system, and lab result feeds.

The Chief Medical Officer cannot answer a simple question like "What is our average ER wait time by day of week and staffing level?" without manually querying three systems. The current state:

- **CMS compliance risk:** Readmission rate reporting takes 2 analysts 3 weeks per quarter. CMS requires accurate submission within 45 days of quarter end — MedVista is consistently late
- **Operational inefficiency:** Bed management is done via whiteboards and phone calls — no real-time visibility into available beds across the 12-hospital network
- **Supply cost overruns:** Supply chain costs increased 23% year-over-year but nobody can correlate supply usage with patient volume to find waste
- **Microsoft ecosystem fit:** MedVista already uses Microsoft 365, Azure Active Directory, and Power BI — Fabric is the natural extension with no new vendor relationship

**Target:** A unified Fabric analytics platform with Bronze/Silver/Gold medallion architecture, daily automated pipelines, a Fabric Warehouse for SQL analytics, and a real-time bed utilization dashboard via Eventstreams and KQL.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why three separate Lakehouses (not one)?** | Isolation and security. Raw PHI data in Bronze requires stricter access controls than analytics-ready Gold data. Data quality issues in Bronze (malformed EHR exports) don't propagate to Gold reports. Each layer can be reprocessed independently. |
| **Why Delta MERGE for Silver instead of overwrite?** | Encounter records can be updated hours or days after initial admission (discharge time added, final diagnosis coded). MERGE handles late updates without reloading the full table. Overwrite would lose the previous day's updates. |
| **Why a Fabric Warehouse in addition to the Gold Lakehouse?** | The CMO's analysts know T-SQL but not PySpark. The Warehouse exposes the Gold Lakehouse tables via shortcuts as T-SQL-queryable objects. Analytical views in T-SQL are easier to maintain than DAX measures for complex SQL logic. |
| **Why Eventstreams + KQL for bed utilization (not just batch)?** | Bed availability changes every few minutes during peak hours. A daily batch refresh means the dashboard shows bed counts that are 12 hours stale. KQL queries on Eventstream data are current to within 10 seconds. |

---

## WHAT YOU WILL BUILD

```
MedVista Fabric Platform:
│
├── 1. WORKSPACES & LAKEHOUSES
│   ├── medvista-data-engineering (workspace)
│   │   ├── medvista_bronze  (Lakehouse — raw source data)
│   │   ├── medvista_silver  (Lakehouse — cleaned, conformed)
│   │   └── medvista_gold    (Lakehouse — star schema)
│   └── medvista-analytics (workspace)
│       ├── medvista_warehouse  (Fabric Warehouse)
│       └── MedVista Power BI Reports
│
├── 2. DAILY INGESTION PIPELINE
│   ├── Copy: Epic EHR SFTP → Bronze encounters
│   ├── Copy: Workday REST API → Bronze staff schedules
│   ├── Copy: SAP OData → Bronze supply orders
│   ├── Notebook: Bronze → Silver (dedup, conform, MERGE)
│   ├── Notebook: Silver → Gold (star schema, MERGE)
│   └── On Failure: Teams webhook notification
│
├── 3. SPARK NOTEBOOKS
│   ├── 01_bronze_to_silver.py  (dedup, type enforce, MERGE)
│   └── 02_silver_to_gold.py   (star schema, OPTIMIZE, assertions)
│
├── 4. FABRIC WAREHOUSE VIEWS
│   ├── vw_hospital_performance
│   ├── vw_supply_cost_yoy
│   └── vw_readmission_analysis
│
└── 5. REAL-TIME INTELLIGENCE
    ├── Eventstream: medvista_bed_events (Event Hub → KQL)
    └── KQL Queries: current occupancy, turnover time, alerts
```

### Time Estimate: 10–12 hours

---

## PART 1: WORKSPACE AND LAKEHOUSE SETUP (30 min)

### Step 1.1: Create the Workspace and Lakehouse Structure

In the Fabric portal, create the following (UI steps documented as reference):

```
1. Create workspace: "medvista-data-engineering"
   - Assign to Fabric capacity (trial or F-SKU)
   - Git integration: connect to Azure DevOps repo (optional but recommended)

2. Create workspace: "medvista-analytics"
   - Assign to same capacity

3. In medvista-data-engineering, create 3 Lakehouses:
   - medvista_bronze
   - medvista_silver
   - medvista_gold

4. In medvista-analytics, create 1 Warehouse:
   - medvista_warehouse
```

### Step 1.2: Create Bronze Table Schemas

```python
# Fabric Notebook: 00_create_bronze_schemas.py
# Run once to establish Bronze table structures.

# WHY explicit schema: prevents Bronze from auto-inferring wrong types
# from malformed source data. A numeric field with one "N/A" value
# would be inferred as STRING without explicit schema.

spark.sql("""
    CREATE TABLE IF NOT EXISTS medvista_bronze.ehr_patient_encounters (
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
        _file_name          STRING
    )
    USING DELTA
    PARTITIONED BY (DATE(_ingested_at))
""")

spark.sql("""
    CREATE TABLE IF NOT EXISTS medvista_bronze.hr_staff_schedules (
        schedule_id     STRING,
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
        _file_name      STRING
    )
    USING DELTA
    PARTITIONED BY (shift_date)
""")

spark.sql("""
    CREATE TABLE IF NOT EXISTS medvista_bronze.supply_orders (
        order_id        STRING,
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
        _ingested_at    TIMESTAMP
    )
    USING DELTA
    PARTITIONED BY (order_date)
""")

print("Bronze schemas created")
```

---

## PART 2: FABRIC PIPELINE — DAILY INGESTION (45 min)

### Step 2.1: Pipeline Parameter and Activity Configuration

The pipeline is built in the Fabric Pipeline designer. The JSON below documents each activity configuration for reference and version control.

```json
{
  "name": "MedVista_Daily_Ingestion",
  "parameters": {
    "processDate": {
      "type": "string",
      "defaultValue": "@formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')"
    }
  },
  "activities": [
    {
      "name": "Copy_EHR_Encounters",
      "type": "Copy",
      "source": {
        "type": "DelimitedText",
        "connection": "MedVista_EHR_SFTP",
        "filePath": "@concat('exports/encounters_', pipeline().parameters.processDate, '.csv')",
        "firstRowAsHeader": true
      },
      "sink": {
        "type": "LakehouseTable",
        "lakehouse": "medvista_bronze",
        "tableName": "ehr_patient_encounters",
        "tableActionOption": "Append"
      },
      "mapping": {
        "additionalColumns": [
          {"name": "_source_system", "value": "epic_ehr"},
          {"name": "_ingested_at",   "value": "@utcNow()"},
          {"name": "_file_name",     "value": "@concat('encounters_', pipeline().parameters.processDate, '.csv')"}
        ]
      },
      "retryCount": 3,
      "retryIntervalInSeconds": 60
    },
    {
      "name": "Copy_HR_Schedules",
      "type": "Copy",
      "dependsOn": [],
      "source": {
        "type": "RestSource",
        "connection": "MedVista_Workday_API",
        "relativeUrl": "@concat('/api/v1/schedules?date=', pipeline().parameters.processDate)",
        "requestMethod": "GET",
        "paginationRules": {"nextPageUrl": "$.next_page_url"}
      },
      "sink": {
        "type": "LakehouseTable",
        "lakehouse": "medvista_bronze",
        "tableName": "hr_staff_schedules",
        "tableActionOption": "Append"
      }
    },
    {
      "name": "Copy_Supply_Orders",
      "type": "Copy",
      "dependsOn": [],
      "source": {
        "type": "OData",
        "connection": "MedVista_SAP_OData",
        "path": "/sap/opu/odata/sap/SUPPLY_SRV/Orders",
        "query": "@concat('$filter=OrderDate eq datetime''', pipeline().parameters.processDate, 'T00:00:00''')"
      },
      "sink": {
        "type": "LakehouseTable",
        "lakehouse": "medvista_bronze",
        "tableName": "supply_orders",
        "tableActionOption": "Append"
      }
    },
    {
      "name": "Transform_Bronze_to_Silver",
      "type": "Notebook",
      "dependsOn": [
        {"activity": "Copy_EHR_Encounters",  "dependencyConditions": ["Succeeded"]},
        {"activity": "Copy_HR_Schedules",    "dependencyConditions": ["Succeeded"]},
        {"activity": "Copy_Supply_Orders",   "dependencyConditions": ["Succeeded"]}
      ],
      "notebookReference": "01_bronze_to_silver",
      "parameters": {"process_date": "@pipeline().parameters.processDate"}
    },
    {
      "name": "Transform_Silver_to_Gold",
      "type": "Notebook",
      "dependsOn": [
        {"activity": "Transform_Bronze_to_Silver", "dependencyConditions": ["Succeeded"]}
      ],
      "notebookReference": "02_silver_to_gold",
      "parameters": {"process_date": "@pipeline().parameters.processDate"}
    },
    {
      "name": "Alert_On_Failure",
      "type": "WebActivity",
      "dependsOn": [
        {"activity": "Transform_Silver_to_Gold", "dependencyConditions": ["Failed"]},
        {"activity": "Transform_Bronze_to_Silver", "dependencyConditions": ["Failed"]},
        {"activity": "Copy_EHR_Encounters", "dependencyConditions": ["Failed"]}
      ],
      "url": "https://outlook.office.com/webhook/TEAMS_WEBHOOK_URL",
      "method": "POST",
      "body": {
        "text": "@concat('⚠️ MedVista Pipeline FAILED\\nDate: ', pipeline().parameters.processDate, '\\nPipeline Run: ', pipeline().RunId)"
      }
    }
  ]
}
```

---

## PART 3: BRONZE TO SILVER NOTEBOOK (60 min)

### Step 3.1: Complete Transformation Notebook

```python
# Fabric Notebook: 01_bronze_to_silver.py
# Transforms Bronze raw data into cleaned, conformed Silver tables.
# Called by the daily pipeline with process_date parameter.

from pyspark.sql import functions as F
from pyspark.sql.window import Window
from delta.tables import DeltaTable
import json

# ── Parameters ───────────────────────────────────────────────────────────────
process_date = dbutils.widgets.get("process_date")   # "2026-03-19"
print(f"Bronze → Silver transformation for: {process_date}")

run_log = {"process_date": process_date, "tables": {}}


# ════════════════════════════════════════════════════════════════════
# TABLE 1: Patient Encounters
# ════════════════════════════════════════════════════════════════════

bronze_enc = (
    spark.read.format("delta")
    .table("medvista_bronze.ehr_patient_encounters")
    .filter(F.col("_ingested_at").cast("date") == process_date)
)
raw_count = bronze_enc.count()

silver_enc = (
    bronze_enc
    # Deduplicate — same encounter may arrive in multiple daily files
    .dropDuplicates(["encounter_id"])

    # Standardise encounter types across EHR source variations
    .withColumn("encounter_type",
        F.when(F.upper(F.col("encounter_type")).isin("ER","ED","EMERGENCY"), "EMERGENCY")
         .when(F.upper(F.col("encounter_type")).isin("IP","INPATIENT"),      "INPATIENT")
         .when(F.upper(F.col("encounter_type")).isin("OP","OUTPATIENT"),     "OUTPATIENT")
         .when(F.upper(F.col("encounter_type")).isin("OBS","OBSERVATION"),   "OBSERVATION")
         .otherwise(F.upper(F.trim(F.col("encounter_type"))))
    )

    # Calculate length of stay — requires valid discharge time
    .withColumn("length_of_stay_hours",
        F.when(F.col("discharge_datetime").isNotNull(),
            F.round(
                (F.unix_timestamp("discharge_datetime") -
                 F.unix_timestamp("admission_datetime")) / 3600.0, 2
            )
        ).otherwise(None)
    )

    # CMS readmission flag: same patient admitted within 30 days of prior discharge
    .withColumn("is_readmission",
        F.when(
            F.col("admission_datetime") <= (
                F.lag("discharge_datetime").over(
                    Window.partitionBy("patient_id")
                    .orderBy("admission_datetime")
                ) + F.expr("INTERVAL 30 DAYS")
            ),
            True
        ).otherwise(False)
    )

    # Data validity filter: admission must precede discharge
    .filter(
        (F.col("admission_datetime") < F.col("discharge_datetime")) |
        F.col("discharge_datetime").isNull()  # Still admitted — valid
    )

    # Null policy: insurance_type unknown = "UNKNOWN" (not null)
    .withColumn("insurance_type",
        F.coalesce(F.col("insurance_type"), F.lit("UNKNOWN"))
    )
    # Validity: charges must be non-negative
    .withColumn("total_charges",
        F.when(F.col("total_charges") < 0, 0.0).otherwise(F.col("total_charges"))
    )

    .withColumn("_processed_at", F.current_timestamp())
    .withColumn("_process_date",  F.lit(process_date))
    .drop("_source_system", "_ingested_at", "_file_name")
)

clean_count = silver_enc.count()
reject_count = raw_count - clean_count

# Upsert (MERGE) into Silver
if DeltaTable.isDeltaTable(spark, "Tables/silver_encounters"):
    silver_table = DeltaTable.forName(spark, "medvista_silver.silver_encounters")
    (silver_table.alias("t")
     .merge(silver_enc.alias("s"), "t.encounter_id = s.encounter_id")
     .whenMatchedUpdate(
         condition="s.discharge_datetime IS NOT NULL",
         set={"discharge_datetime":   "s.discharge_datetime",
              "length_of_stay_hours": "s.length_of_stay_hours",
              "total_charges":        "s.total_charges",
              "_processed_at":        "s._processed_at"}
     )
     .whenNotMatchedInsertAll()
     .execute()
    )
else:
    (silver_enc.write.format("delta").mode("overwrite")
     .option("overwriteSchema", "true")
     .partitionBy("encounter_type")
     .saveAsTable("medvista_silver.silver_encounters"))

run_log["tables"]["silver_encounters"] = {
    "raw": raw_count, "clean": clean_count, "rejected": reject_count
}
print(f"  Encounters — raw: {raw_count:,} | clean: {clean_count:,} | rejected: {reject_count:,}")


# ════════════════════════════════════════════════════════════════════
# TABLE 2: Staff Schedules
# ════════════════════════════════════════════════════════════════════

bronze_sched = (
    spark.read.format("delta")
    .table("medvista_bronze.hr_staff_schedules")
    .filter(F.col("shift_date") == process_date)
    .dropDuplicates(["schedule_id"])
)

silver_sched = (
    bronze_sched
    .withColumn("shift_hours",
        F.round((F.unix_timestamp("shift_end") - F.unix_timestamp("shift_start")) / 3600.0, 2)
    )
    .withColumn("role_category",
        F.when(F.col("role").rlike("(?i)physician|doctor|md|do"), "PHYSICIAN")
         .when(F.col("role").rlike("(?i)nurse|rn|lpn|np"),        "NURSING")
         .when(F.col("role").rlike("(?i)tech|aide|assistant"),     "SUPPORT")
         .otherwise("OTHER")
    )
    .withColumn("is_abnormal_shift",
        (F.col("shift_hours") > 16) | (F.col("shift_hours") < 2)
    )
    .withColumn("_processed_at", F.current_timestamp())
    .drop("_source_system", "_ingested_at", "_file_name")
)

(silver_sched.write.format("delta").mode("overwrite")
 .option("replaceWhere", f"shift_date = '{process_date}'")
 .saveAsTable("medvista_silver.silver_staff_schedules"))

run_log["tables"]["silver_staff_schedules"] = {"written": silver_sched.count()}


# ════════════════════════════════════════════════════════════════════
# TABLE 3: Supply Orders
# ════════════════════════════════════════════════════════════════════

bronze_supply = (
    spark.read.format("delta")
    .table("medvista_bronze.supply_orders")
    .filter(F.col("order_date") == process_date)
    .dropDuplicates(["order_id"])
)

silver_supply = (
    bronze_supply
    .withColumn("category_group",
        F.when(F.col("category").rlike("(?i)pharma|drug|medication"), "PHARMACEUTICALS")
         .when(F.col("category").rlike("(?i)surgical|implant"),        "SURGICAL")
         .when(F.col("category").rlike("(?i)ppe|glove|mask|gown"),     "PPE")
         .when(F.col("category").rlike("(?i)lab|test|reagent"),        "LABORATORY")
         .otherwise("GENERAL")
    )
    # Derive total_cost if missing (prevent null measure in Gold)
    .withColumn("total_cost",
        F.coalesce(F.col("total_cost"), F.col("quantity") * F.col("unit_cost"))
    )
    .filter(F.col("total_cost") > 0)
    .withColumn("_processed_at", F.current_timestamp())
    .drop("_source_system", "_ingested_at")
)

(silver_supply.write.format("delta").mode("overwrite")
 .option("replaceWhere", f"order_date = '{process_date}'")
 .saveAsTable("medvista_silver.silver_supply_orders"))

run_log["tables"]["silver_supply_orders"] = {"written": silver_supply.count()}

print(f"\nBronze → Silver complete. Summary: {json.dumps(run_log, indent=2)}")
dbutils.notebook.exit(json.dumps(run_log))
```

---

## PART 4: SILVER TO GOLD NOTEBOOK (60 min)

### Step 4.1: Star Schema Construction with OPTIMIZE and Assertions

```python
# Fabric Notebook: 02_silver_to_gold.py
# Builds the Gold star schema from Silver conformed tables.
# Includes OPTIMIZE, ZORDER, and post-load assertions.

from pyspark.sql import functions as F
from delta.tables import DeltaTable
import json

process_date = dbutils.widgets.get("process_date")
date_key_int = int(process_date.replace("-", ""))
print(f"Silver → Gold for: {process_date}")


# ════════════════════════════════════════════════════════════════════
# DIMENSION: dim_hospital (static — full reload)
# ════════════════════════════════════════════════════════════════════

dim_hospital = spark.createDataFrame([
    ("H001","MedVista Downtown",   "Metro City",  "East",  450,"Level I Trauma"),
    ("H002","MedVista Northside",  "Northville",  "North", 320,"Community"),
    ("H003","MedVista Eastside",   "Eastport",    "East",  280,"Community"),
    ("H004","MedVista Southgate",  "Southdale",   "South", 200,"Critical Access"),
    ("H005","MedVista West Campus","Westfield",   "West",  380,"Teaching"),
    ("H006","MedVista Children's", "Metro City",  "East",  150,"Pediatric Specialty"),
    ("H007","MedVista Rehab",      "Greenwood",   "North", 100,"Rehabilitation"),
    ("H008","MedVista Behavioral", "Lakeside",    "West",   80,"Psychiatric"),
    ("H009","MedVista Heart Ctr",  "Metro City",  "East",  120,"Cardiac Specialty"),
    ("H010","MedVista Surgical",   "Riverside",   "South", 160,"Ambulatory Surgical"),
    ("H011","MedVista Urgent N",   "Northville",  "North",  30,"Urgent Care"),
    ("H012","MedVista Urgent S",   "Southdale",   "South",  30,"Urgent Care"),
], schema="hospital_id STRING, hospital_name STRING, city STRING, region STRING, bed_count INT, hospital_type STRING")

dim_hospital = dim_hospital.withColumn(
    "hospital_key", F.monotonically_increasing_id() + 1
)

(dim_hospital.write.format("delta").mode("overwrite")
 .saveAsTable("medvista_gold.dim_hospital"))


# ════════════════════════════════════════════════════════════════════
# FACT: fact_encounters (incremental MERGE on encounter_id)
# ════════════════════════════════════════════════════════════════════

silver_enc = (
    spark.read.format("delta")
    .table("medvista_silver.silver_encounters")
    .filter(F.col("_process_date") == process_date)
)

fact_enc = (
    silver_enc
    .join(dim_hospital.select("hospital_id", "hospital_key"),
          on="hospital_id", how="left")
    .withColumn("admission_date_key",
        F.date_format("admission_datetime", "yyyyMMdd").cast("int")
    )
    .withColumn("discharge_date_key",
        F.when(F.col("discharge_datetime").isNotNull(),
               F.date_format("discharge_datetime", "yyyyMMdd").cast("int"))
    )
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

if DeltaTable.isDeltaTable(spark, "Tables/fact_encounters"):
    gold_fact = DeltaTable.forName(spark, "medvista_gold.fact_encounters")
    (gold_fact.alias("t")
     .merge(fact_enc.alias("s"), "t.encounter_id = s.encounter_id")
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())
else:
    (fact_enc.write.format("delta").mode("overwrite")
     .saveAsTable("medvista_gold.fact_encounters"))

enc_count = fact_enc.count()
print(f"fact_encounters: {enc_count:,} records for {process_date}")


# ════════════════════════════════════════════════════════════════════
# FACT: fact_supply_costs (partition-scoped overwrite)
# ════════════════════════════════════════════════════════════════════

silver_supply = (
    spark.read.format("delta")
    .table("medvista_silver.silver_supply_orders")
    .filter(F.col("order_date") == process_date)
)

fact_supply = (
    silver_supply
    .join(dim_hospital.select("hospital_id", "hospital_key"),
          on="hospital_id", how="left")
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

(fact_supply.write.format("delta").mode("overwrite")
 .option("replaceWhere", f"date_key = {date_key_int}")
 .saveAsTable("medvista_gold.fact_supply_costs"))


# ════════════════════════════════════════════════════════════════════
# POST-LOAD QUALITY ASSERTIONS
# Pipeline FAILS immediately if any assertion fails — no silent bad data.
# ════════════════════════════════════════════════════════════════════

# Assertion 1: Encounter count is plausible (at least 100, at most 50K per day)
assert 100 <= enc_count <= 50_000, \
    f"Encounter count {enc_count} outside expected range [100, 50000]"

# Assertion 2: No null hospital_key in fact (would break all hospital-filtered reports)
null_hospital_keys = fact_enc.filter(F.col("hospital_key").isNull()).count()
assert null_hospital_keys == 0, \
    f"{null_hospital_keys} encounter records have null hospital_key — missing dim_hospital entry?"

# Assertion 3: No negative total_charges
neg_charges = fact_enc.filter(F.col("total_charges") < 0).count()
assert neg_charges == 0, \
    f"{neg_charges} encounter records have negative total_charges"

# Assertion 4: Readmission rate is plausible (< 30%)
total_enc = spark.read.format("delta").table("medvista_gold.fact_encounters").count()
readmit_count = spark.read.format("delta").table("medvista_gold.fact_encounters") \
    .filter(F.col("is_readmission") == True).count()
readmit_rate = readmit_count / total_enc
assert readmit_rate < 0.30, \
    f"Readmission rate {readmit_rate:.1%} exceeds 30% — data quality issue?"

print("All assertions passed ✓")


# ════════════════════════════════════════════════════════════════════
# OPTIMIZE AND ZORDER
# Run after MERGE to compact small files created by the MERGE operation.
# ZORDER by the most common query filter columns.
# ════════════════════════════════════════════════════════════════════

spark.sql("""
    OPTIMIZE medvista_gold.fact_encounters
    ZORDER BY (hospital_key, admission_date_key)
""")
# WHY these columns: 90% of CMO dashboard queries filter by hospital and date range.
# ZORDER co-locates related data → data-skipping reduces files scanned 5-10x.

spark.sql("""
    OPTIMIZE medvista_gold.fact_supply_costs
    ZORDER BY (hospital_key, date_key)
""")

print(f"\nSilver → Gold complete for {process_date}")
dbutils.notebook.exit(json.dumps({
    "process_date": process_date,
    "fact_encounters_written": enc_count,
    "assertions": "passed"
}))
```

---

## PART 5: FABRIC WAREHOUSE VIEWS (45 min)

### Step 5.1: Create Shortcuts and Analytical Views

In the Fabric Warehouse (medvista_warehouse), first create shortcuts to the Gold Lakehouse tables, then build analytical views:

```sql
-- ════════════════════════════════════════════════════════════════════
-- STEP 1: Create shortcuts to Gold Lakehouse tables
-- (Done in the Warehouse UI: New Shortcut → OneLake → medvista_gold)
-- Tables to shortcut: fact_encounters, fact_supply_costs, dim_hospital
-- ════════════════════════════════════════════════════════════════════

-- ════════════════════════════════════════════════════════════════════
-- VIEW 1: Hospital Performance Summary
-- Used by: CMO monthly dashboard, CMS compliance reports
-- ════════════════════════════════════════════════════════════════════

CREATE VIEW analytics.vw_hospital_performance AS
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
    PERCENTILE_CONT(0.5) WITHIN GROUP
        (ORDER BY f.length_of_stay_hours)                   AS median_los_hours,

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

FROM medvista_gold.fact_encounters   f   -- shortcut to Gold Lakehouse
JOIN medvista_gold.dim_hospital      h ON f.hospital_key = h.hospital_key
JOIN medvista_gold.dim_date          d ON f.admission_date_key = d.date_key
GROUP BY
    h.hospital_name, h.region, h.hospital_type, h.bed_count,
    d.full_date, d.month_name, d.year_number, d.quarter_name;


-- ════════════════════════════════════════════════════════════════════
-- VIEW 2: Supply Cost Year-over-Year Analysis
-- Used by: CFO supply cost dashboard, contract negotiation prep
-- ════════════════════════════════════════════════════════════════════

CREATE VIEW analytics.vw_supply_cost_yoy AS
WITH monthly AS (
    SELECT
        h.hospital_name, h.region, s.category_group,
        d.year_number, d.month_number, d.month_name,
        SUM(s.total_cost)    AS monthly_cost,
        SUM(s.total_quantity) AS monthly_quantity
    FROM medvista_gold.fact_supply_costs s
    JOIN medvista_gold.dim_hospital      h ON s.hospital_key = h.hospital_key
    JOIN medvista_gold.dim_date          d ON s.date_key = d.date_key
    GROUP BY h.hospital_name, h.region, s.category_group,
             d.year_number, d.month_number, d.month_name
),
with_lag AS (
    SELECT *,
        LAG(monthly_cost) OVER (
            PARTITION BY hospital_name, category_group, month_number
            ORDER BY year_number
        ) AS prior_year_cost
    FROM monthly
)
SELECT *,
    CASE
        WHEN prior_year_cost IS NOT NULL AND prior_year_cost > 0
        THEN ROUND((monthly_cost - prior_year_cost) / prior_year_cost * 100, 2)
        ELSE NULL
    END AS yoy_cost_change_pct
FROM with_lag;
```

---

## PART 6: REAL-TIME INTELLIGENCE (45 min)

### Step 6.1: Eventstream Configuration

In the Fabric portal:
1. Create Eventstream "medvista_bed_events"
2. Source: Azure Event Hub (bed management system publishes ADMIT/DISCHARGE/CLEAN/MAINTENANCE events)
3. Transformation: Filter (no transformation — route all events)
4. Destination 1: KQL Database "medvista_realtime", table "bed_events"
5. Destination 2: Lakehouse "medvista_gold", table "bed_events_archive" (long-term storage)

### Step 6.2: KQL Queries for Real-Time Dashboards

```kql
// ── Query 1: Current bed occupancy across all hospitals ──────────────────────
// arg_max returns the most recent event_type per bed — current status
bed_events
| where event_timestamp > ago(24h)
| summarize arg_max(event_timestamp, event_type) by bed_id, ward_id, hospital_id
| summarize
    total_beds    = dcount(bed_id),
    occupied      = dcountif(bed_id, event_type == "ADMIT"),
    available     = dcountif(bed_id, event_type == "CLEAN"),
    being_cleaned = dcountif(bed_id, event_type == "DISCHARGE"),
    maintenance   = dcountif(bed_id, event_type == "MAINTENANCE")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total_beds) * 100, 1)
| order by occupancy_pct desc
| render barchart with (title="Current Bed Occupancy by Hospital")


// ── Query 2: Hourly occupancy trend (24 hours) ───────────────────────────────
bed_events
| where event_timestamp > ago(24h) and event_type == "ADMIT"
| summarize admits = count() by hospital_id, hour = bin(event_timestamp, 1h)
| render timechart with (title="Admissions per Hour")


// ── Query 3: Bed turnover time (discharge → next admit) ──────────────────────
// WHY: Faster turnaround = more effective capacity without adding beds
bed_events
| where event_timestamp > ago(7d)
| where event_type in ("DISCHARGE","ADMIT")
| sort by bed_id, event_timestamp asc
| extend next_event = next(event_type), next_time = next(event_timestamp)
| where event_type == "DISCHARGE" and next_event == "ADMIT"
| extend turnover_minutes = datetime_diff('minute', next_time, event_timestamp)
| summarize
    avg_turnover_min = round(avg(turnover_minutes), 0),
    p90_turnover_min = round(percentile(turnover_minutes, 90), 0)
    by hospital_id, bin(event_timestamp, 1d)
| render timechart with (title="Daily Average Bed Turnover Time (Minutes)")


// ── Query 4: Alert — hospitals approaching capacity (>= 90%) ─────────────────
// Use this as a dashboard tile that shows RED when hospitals are near capacity
bed_events
| where event_timestamp > ago(1h)
| summarize arg_max(event_timestamp, event_type) by bed_id, hospital_id
| summarize
    total    = dcount(bed_id),
    occupied = dcountif(bed_id, event_type == "ADMIT")
    by hospital_id
| extend occupancy_pct = round(todouble(occupied) / todouble(total) * 100, 1)
| where occupancy_pct >= 85
| project hospital_id, occupancy_pct, occupied, total,
    available_beds = total - occupied,
    alert_level = case(
        occupancy_pct >= 95, "CRITICAL 🔴",
        occupancy_pct >= 90, "WARNING 🟡",
        "WATCH 🟠"
    )
| order by occupancy_pct desc
```

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The Bronze-to-Silver notebook uses `dropDuplicates(["encounter_id"])` before the Delta MERGE. If instead you ran the MERGE without deduplication, and today's Bronze file contained the same encounter_id twice with different discharge times, what would happen in the MERGE? Why does Delta MERGE behave this way, and how would you fix it?

2. The Silver-to-Gold notebook uses `OPTIMIZE ... ZORDER BY (hospital_key, admission_date_key)`. After a week of daily pipelines, you notice that queries filtering on `encounter_type` (e.g., "show me all EMERGENCY encounters") are slow. Would adding `encounter_type` to the ZORDER clause help? What is the practical limit to how many columns you should include in ZORDER, and why?

3. The bed events KQL query uses `where event_timestamp > ago(24h)` before `arg_max`. A bed in Hospital H002 has had no events for 26 hours (the patient has been in it for 3 days without a status update). What does the current occupancy query return for that bed, and what change to the KQL would produce a more accurate result?

4. The pipeline has three parallel Copy activities (EHR, HR Schedules, Supply Orders) that all complete before the Silver notebook starts. The EHR Copy takes 45 minutes, HR takes 5 minutes, and Supply takes 8 minutes. If you need to reduce the total pipeline duration, which activity should you focus on, and what are two Fabric-native approaches to reducing its run time?

5. The CMO asks you to add patient age and gender to `fact_encounters` for demographic analysis. Currently, age and gender are in the Epic EHR source but are not ingested. Walk through the exact changes needed at each layer: Bronze table schema change, Silver transformation addition, Gold MERGE update, Warehouse view update, and Power BI semantic model update. Which step has the highest risk of breaking existing downstream artefacts?
