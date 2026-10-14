# Lab 04 — Silver to Gold: Star Schema and Delta Maintenance

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Goal:** Build `02_silver_to_gold`: a date dimension, a hospital dimension with a stable key, an incrementally merged encounter fact, a daily supply-cost fact, post-load quality assertions, and OPTIMIZE/ZORDER maintenance.
> **Time:** 3–4 hours | **Difficulty:** Intermediate | **Prerequisites:** [Lab_03](Lab_03_Bronze_to_Silver_PySpark_and_MERGE.md) — `silver_encounters` has 3,119 rows; Silver schedules and supply loaded for all three dates
> **Guided lab coverage:** Part 4, Step 4.1 (complete Silver → Gold notebook)
> **Cost:** $0 on the trial. OPTIMIZE is a Spark job like any other; stop the session at the end.

---

## Learning objectives

By the end of this lab you can:

- Design a star schema: choose each fact's grain, and its dimensions and measures.
- Build a date dimension with `sequence()` and `explode()`.
- Explain why `monotonically_increasing_id()` is not a surrogate key, and derive a stable one.
- Load a fact incrementally with MERGE and a daily aggregate with a partition-scoped overwrite.
- Write post-load assertions that fail the notebook (and later the pipeline) on bad data.
- Use `DESCRIBE DETAIL`, `OPTIMIZE ... ZORDER BY` and `VACUUM`, and explain V-Order.

---

## 1. Environment Setup

Verify Lab_03:

```python
display(spark.sql("""SELECT _process_date, COUNT(*) AS rows
                     FROM medvista_silver.dbo.silver_encounters
                     GROUP BY _process_date ORDER BY _process_date"""))
```

**Expected:** 2026-03-17 → 844, 2026-03-18 → 1,027, 2026-03-19 → 1,248 (3,119 in total). Day 17 shows fewer than the 1,041 rows it inserted because 197 of them were updated by later late-discharge MERGEs, which moved their `_process_date` forward.

Create notebook `02_silver_to_gold`. In its Explorer, add **medvista_gold** (pin it as default) and **medvista_silver**. Cell 1, marked as the parameter cell:

```python
process_date = "2026-03-17"
```

---

## 2. Business Context

The CMO's analysts don't want 20 columns of encounter detail; they want "encounters, readmission rate and average length of stay **by hospital, by month**", and the CFO wants "supply cost **by hospital, category and day**". A star schema gives them exactly that shape: narrow fact tables of measures, joined to small dimension tables they can filter and group by. Lab_06 builds T-SQL views on top; Power BI's Direct Lake mode reads the same Delta tables.

Gold is also the last gate before people make decisions, so it carries assertions: if today's load looks wrong, the pipeline fails and nobody sees the numbers.

---

## 3. Concept Explanation

### 3.1 The MedVista star schema

```
                 dim_date (date_key)                      dim_hospital (hospital_key)
                      │                                           │
   ┌──────────────────┼──────────────────┐        ┌───────────────┴──────────┐
fact_encounters                          fact_supply_costs
grain: one row per encounter             grain: one row per hospital × department × category × day
keys: hospital_key, admission_date_key,  keys: hospital_key, date_key
      discharge_date_key                 measures: order_count, total_quantity, total_cost,
measures: length_of_stay_hours,                    avg_unit_cost, unique_items, unique_vendors
          total_charges, is_readmission
```

### 3.2 Surrogate keys that don't move

A surrogate key must mean the same thing on every run. `monotonically_increasing_id()` only guarantees *unique and increasing within one run*: its values encode the partition number, so a reload with a different partitioning gives hospitals new keys — and every existing fact row silently points at the wrong hospital. For a small reference dimension with a clean business key (`H001` … `H012`), derive the key from it: `H007` → `7`, every time.

### 3.3 Two load patterns for two facts

| Fact | Changes after first load? | Pattern |
|---|---|---|
| `fact_encounters` | Yes — discharges arrive late | MERGE on `encounter_id`, reading Silver rows with `_process_date = today` |
| `fact_supply_costs` | No — a day's orders are final | Aggregate the day, `replaceWhere date_key = <day>` |

### 3.4 Assertions

A Python `assert` that fails raises `AssertionError`; the notebook fails; in Lab_05 the pipeline's failure path fires. Assertions check things that would silently corrupt reports: implausible volume, orphan keys, negative money, implausible rates.

### 3.5 Small files, OPTIMIZE, ZORDER, VACUUM and V-Order

- Every MERGE and daily write adds files. Many small files make reads slow. **OPTIMIZE** rewrites them into fewer, larger files.
- **ZORDER BY (cols)** clusters rows by those columns while rewriting, so a query filtering on them can skip whole files using min/max statistics. Use 1–2 columns that most queries filter on; each extra column dilutes the others.
- OPTIMIZE doesn't delete the old files (time travel still needs them). **VACUUM** removes files no longer referenced and older than the retention period (default 7 days). Never shorten that casually: you lose time travel and can break long-running readers.
- **V-Order** is a Fabric-specific Parquet write optimisation that speeds up Power BI Direct Lake and SQL endpoint reads. Whether it's on by default depends on your workspace's Spark settings; check `spark.conf.get("spark.sql.parquet.vorder.default")`.

---

## 4. Step-by-Step Implementation

### Step 1 — Build and inspect `dim_date`

```python
from pyspark.sql import functions as F
dim_date = (spark.sql("SELECT explode(sequence(to_date('2024-01-01'), to_date('2027-12-31'), interval 1 day)) AS full_date")
    .select(F.date_format("full_date", "yyyyMMdd").cast("int").alias("date_key"), "full_date",
            F.year("full_date").alias("year_number"),
            F.concat(F.year("full_date").cast("string"), F.lit("-Q"), F.quarter("full_date").cast("string")).alias("quarter_name"),
            F.month("full_date").alias("month_number"),
            F.date_format("full_date", "MMMM").alias("month_name"),
            F.date_format("full_date", "EEEE").alias("day_name"),
            F.dayofweek("full_date").isin(1, 7).alias("is_weekend")))
print(dim_date.count())
display(dim_date.filter("full_date = '2026-03-17'"))
```

**Expected output:** `1461`, and one row: `20260317 | 2026-03-17 | 2026 | 2026-Q1 | 3 | March | Tuesday | false`.

### Step 2 — See why `monotonically_increasing_id()` isn't a key

```python
hosp = spark.createDataFrame([(f"H{i:03d}",) for i in range(1, 13)], "hospital_id STRING")
display(hosp.repartition(3).withColumn("unstable_key", F.monotonically_increasing_id())
            .withColumn("stable_key", F.expr("CAST(substring(hospital_id, 2) AS INT)"))
            .orderBy("hospital_id"))
```

**Expected output:** `unstable_key` values like `0`, `1`, `8589934592`, `17179869184` — they jump by 2³³ between partitions, and which hospital gets which value changes if you run it again with a different number of partitions. `stable_key` is 1–12, always.

✅ **Checkpoint:** you can explain why an unstable dimension key is a silent data-corruption bug in every fact table that references it.

### Step 3 — The complete notebook

Replace everything after the parameter cell with the guided lab's notebook:

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

### Step 4 — Run three days, in order

Run all with `process_date = "2026-03-17"`, then `2026-03-18`, then `2026-03-19`.

**Expected output (you ran Silver for all three days first in Lab_03, so each Gold run picks up the rows whose latest change was on that date):**

| process_date | `fact_encounters` written | fact total | Readmission rate | `fact_supply_costs` rows for the day |
|---|---|---|---|---|
| 2026-03-17 | 844 | 844 | 5.7% | 255 |
| 2026-03-18 | 1,027 | 1,871 | 7.0% | 251 |
| 2026-03-19 | 1,248 | 3,119 | 7.9% | 249 |

The first run also prints `dim_date created`. Each run ends with `All assertions passed (readmission rate …)`.

The guided lab's pipeline processes Silver *and* Gold for each date before moving on, which gives 1,041 / 1,173 / 1,248 per day instead. The final state — 3,119 encounters, 7.9% readmissions — is the same either way. Lab_05 rebuilds from Bronze and shows you those numbers.

✅ **Checkpoint:**

```python
for t in ["dim_date", "dim_hospital", "fact_encounters", "fact_supply_costs"]:
    print(t, spark.table(f"medvista_gold.dbo.{t}").count())
```

→ `dim_date 1461`, `dim_hospital 12`, `fact_encounters 3119`, `fact_supply_costs 755`.

### Step 5 — Make an assertion fail on purpose

```python
gold_fact = spark.table("medvista_gold.dbo.fact_encounters")
rate = gold_fact.filter("is_readmission").count() / gold_fact.count()
assert rate < 0.05, f"Readmission rate {rate:.1%} exceeds 5% - data quality issue?"
```

**Expected output:** `AssertionError: Readmission rate 7.9% exceeds 5% - data quality issue?` — and the cell is marked failed. In a pipeline, that failure stops the run and triggers the alert (Lab_05). Delete this cell afterwards.

### Step 6 — Look at the files, then OPTIMIZE

The notebook already ran OPTIMIZE at the end of each day. Look at the result and the history:

```python
display(spark.sql("DESCRIBE DETAIL medvista_gold.dbo.fact_encounters").select("numFiles", "sizeInBytes"))
display(spark.sql("DESCRIBE HISTORY medvista_gold.dbo.fact_encounters").select("version", "operation", "operationMetrics"))
```

**Expected output:** a small `numFiles` (typically 1–2 for this data size). History alternates `MERGE` and `OPTIMIZE`. Where an OPTIMIZE had work to do, its metrics show `numRemovedFiles` greater than `numAddedFiles` — several small files rewritten as one. (The first day's OPTIMIZE may report 0 and 0: a single fresh write is already compact.)

Now see what MERGE alone does. Re-run the notebook for `2026-03-19` with the two `OPTIMIZE` lines commented out, then look at the newest `MERGE` row in `DESCRIBE HISTORY`: its `operationMetrics` show `numTargetFilesAdded` — MERGE rewrites the files that contain matched rows and adds new ones, so on a busy table the file count creeps up every day. Uncomment the OPTIMIZE lines and run once more: the new `OPTIMIZE` row compacts them again. Row counts never change — the load is idempotent.

✅ **Checkpoint:** you can show, from `DESCRIBE HISTORY`, that OPTIMIZE reduced the file count without changing the data.

### Step 7 — VACUUM, safely

```python
spark.sql("VACUUM medvista_gold.dbo.fact_encounters RETAIN 168 HOURS")
```

**Expected output:** the command succeeds and removes nothing yet — every unreferenced file is younger than 7 days. Try `RETAIN 1 HOURS`: Delta refuses (retention-duration safety check). Don't disable that check.

### Step 8 — Read the exit value

The notebook ends with `notebookutils.notebook.exit(json.dumps({...}))`. Interactively, the exit value prints under the last cell; in a pipeline it appears in the notebook activity's output, where later activities can read it. Stop the session.

---

## 5. Production Engineering Practices

1. **State the grain of every fact table in a comment.** Most star-schema bugs are grain mismatches (joining day-level supply to encounter-level rows double-counts).
2. **Stable keys or no keys.** Derive keys from business keys, or maintain a key-mapping table; never regenerate keys on reload.
3. **Assert after load, before anyone reads.** The pipeline fails, the alert fires, and the bad numbers stay out of the CMO's dashboard.
4. **Maintenance is part of the load.** OPTIMIZE after MERGE-heavy loads; VACUUM on a schedule with the default (or longer) retention.
5. **ZORDER by what people filter on**, and re-check when query patterns change.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `AssertionError: ... null hospital_key` | A Silver row has a `hospital_id` not in `dim_hospital` | Add the hospital to the reference list, or fix the source code upstream |
| `AssertionError: Encounter count 0 ...` | Silver has no rows with `_process_date` = this date (already picked up, or Silver not run) | Run Silver for that date first; check the Section 1 query |
| `DELTA_UNSUPPORTED...` / syntax error on `OPTIMIZE ... ZORDER` | Not on a Delta table, or a typo in the name | Check the three-part name; `DESCRIBE DETAIL` should say `delta` |
| `requirement failed: Are you sure you would like to vacuum files with such a low retention period?` | Retention below 168 hours | Use `RETAIN 168 HOURS` or more (by design) |
| `fact_supply_costs` doubled for a day | Wrote with `append` instead of `replaceWhere` | Re-run the notebook for that date; it replaces the day |
| `Written data does not conform to partial table overwrite condition` on `fact_supply_costs` | Silver supply rows with `order_date` ≠ `process_date` | The notebook filters on `order_date == process_date`; keep that filter |

---

## 7. Reflection

### Interview questions (with model answers)

1. **"How do you load a fact table whose rows can change after first load?"** — Track which source rows changed in this batch (`_process_date`), and MERGE them on the business key. Insert-only loads miss the changes; full reloads don't scale.
2. **"Why is ZORDER limited to one or two columns in practice?"** — Z-ordering interleaves the sort across columns; each added column weakens clustering on the others, so file skipping gets worse for everyone. Pick the columns most queries filter on.
3. **"OPTIMIZE ran but storage went up. Why?"** — OPTIMIZE writes new compacted files but keeps the old ones for time travel until VACUUM removes them after the retention period.

### Common interview traps

- Using `monotonically_increasing_id()` or `row_number()` over an unordered set as a persistent surrogate key.
- Running VACUUM with zero retention "to save space" and breaking time travel and concurrent readers.

### Key takeaways

- Facts have a stated grain; dimensions have stable keys.
- MERGE for facts that change; partition-scoped overwrite for facts that don't.
- Assertions turn silent bad data into a loud failed run.
- OPTIMIZE/ZORDER after loads; VACUUM with retention on a schedule.

---

## Next

[Lab_05 — Pipeline Orchestration, Alerts and Monitoring](Lab_05_Pipeline_Orchestration_Alerts_and_Monitoring.md): wire Silver and Gold into the pipeline, add the failure path, rebuild Silver and Gold from Bronze, and schedule the daily run.
