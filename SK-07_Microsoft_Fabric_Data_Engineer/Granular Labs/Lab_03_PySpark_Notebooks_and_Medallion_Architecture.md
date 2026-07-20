# Lab 03 — PySpark Notebooks and the Medallion Architecture

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Estimated time:** 4–5 hours
> **Prerequisites:** Lab 02 — `freshcart_bronze` holds `orders_typed` (or `orders_raw`), `products_raw`, `stores_raw`, defects included.

---

## 1. Environment Setup

### 1.1 Create the silver and gold lakehouses

Workspace → **+ New item → Lakehouse**: `freshcart_silver`, then again for `freshcart_gold`.

### 1.2 Create the notebook and attach lakehouses

1. Workspace → **+ New item → Notebook** → rename `01_bronze_to_silver`.
2. Left **Explorer → Add data items → Existing data sources**: attach `freshcart_bronze` AND `freshcart_silver`. Set **`freshcart_silver` as the default lakehouse** (pin icon) — unqualified table writes go to the default; being deliberate here prevents the classic "wrote silver tables into bronze" mess.

### 1.3 Verify Spark works

First cell:

```python
df = spark.read.table("freshcart_bronze.orders_typed")
print(df.count(), "rows")
df.printSchema()
```

Run it (Shift+Enter). **Expected output:** your row count and a typed schema. First run takes 30–90 s (Spark session startup — capacity is allocating your cluster); later cells are fast.

### 1.4 Common problems

| Symptom | Fix |
|---|---|
| `Table or view not found` | Lakehouse not attached, or name/case wrong — check Explorer pane shows both lakehouses |
| Session won't start / capacity throttled | Trial capacity busy — Monitoring hub → check running items; stop stale sessions (they auto-stop after ~20 min idle) |
| Writes land in wrong lakehouse | Default lakehouse mis-set — check the pin, or always fully qualify `lakehouse.table` |

---

## 2. Business Context

FreshCart's bronze data is landed — and it is *dirty by design*: ~0.5% negative quantities (returns keyed as sales), null prices, ~1% duplicate order lines. If those flow into revenue reporting, the numbers are wrong in ways nobody can quantify. The remedy is the **medallion architecture** — the industry's standard layering for lakehouse refinement, and the pattern you already lived in SK-01 (raw/processed/output) and SK-03 (bronze/silver/gold with Spark):

| Layer | FreshCart contents | Contract |
|---|---|---|
| **Bronze** | Tables exactly as landed | Immutable evidence; never edited |
| **Silver** | Cleaned, typed, deduplicated, conformed | Trustworthy atoms; quarantine holds the rejects |
| **Gold** | Star schema: `dim_*` + `fct_*`, aggregates | What BI and the business consume |

**Who consumes what:** engineers debug against bronze; data scientists and Lab 04's warehouse read silver; Lab 06's Power BI reads gold. **If the layering is skipped** ("just clean it in the dashboard"), every report re-implements the cleaning differently, numbers diverge between dashboards, and trust dies — the exact pathology FreshCart hired you to end.

---

## 3. Concept Explanation

### 3.1 Notebooks as production code (not scratchpads)

Fabric notebooks run PySpark against your capacity and are *scheduled as pipeline activities* (Lab 05). So notebook discipline matters: cells in execution order, parameters in a tagged cell at the top, idempotent writes, assertions that fail loudly, and no state that depends on "I ran cell 7 twice." A notebook that only works interactively is a demo; one that runs top-to-bottom cold is a job.

### 3.2 Delta MERGE — idempotency's power tool

`MERGE` upserts: match incoming rows to the target on a key; update matches, insert novelties. Re-running the same batch **changes nothing** (matches update to identical values) — idempotency by construction, no delete-then-insert dance:

```sql
MERGE INTO target t USING updates u ON t.order_id = u.order_id AND t.product_id = u.product_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

This one statement replaces the high-water-mark + delete-scope machinery of earlier modules for most lakehouse loads — know both, prefer MERGE where Delta exists.

### 3.3 Quarantine tables

The dead-letter pattern in Delta form: rows failing validation go to `silver.orders_quarantine` with a `reject_reason` — never dropped. The quarantine's *growth rate* becomes a monitored metric (Lab 05 alerts on it).

### 3.4 Assertions as the notebook's quality gate

`assert` statements (or raising exceptions) after each stage make the *notebook run itself fail* when invariants break — and in Lab 05 a failed notebook fails the pipeline, which alerts the team. Same gate philosophy as SK-01's validator and SK-03's dbt tests, in notebook clothing.

### 3.5 OPTIMIZE and small files

Streaming-ish ingestion and repeated MERGEs create many small Parquet files; queries slow down (file-open overhead dominates). `OPTIMIZE table` compacts them; `VACUUM` removes unreferenced old files after retention. In Fabric, background auto-compaction helps, but knowing *why* small files hurt — and running OPTIMIZE after heavy writes — is expected engineering hygiene.

---

## 4. Step-by-Step Implementation

### Step 1 — Parameterize the notebook

**What:** First code cell — then use the ⋯ menu → **Toggle parameter cell**:

```python
# PARAMETERS (Lab 05's pipeline will override these)
run_date = "2026-06-30"
bronze_lakehouse = "freshcart_bronze"
```

**Why:** parameters are the seam that turns this notebook from "processes everything, always" into "processes the drop for date X" — reruns and backfills become targeted. The tag is what lets pipeline activities inject values.

### Step 2 — Read bronze, profile the damage

**What:**

```python
from pyspark.sql import functions as F

orders = spark.read.table(f"{bronze_lakehouse}.orders_typed")

print("Total rows:", orders.count())
orders.select(
    F.sum(F.when(F.col("quantity") <= 0, 1).otherwise(0)).alias("bad_quantity"),
    F.sum(F.when(F.col("unit_price").isNull(), 1).otherwise(0)).alias("null_price"),
).show()

dupes = orders.groupBy("order_id", "product_id", "order_ts").count().filter("count > 1")
print("Duplicate line groups:", dupes.count())
```

**Expected output:** non-zero counts for all three defect classes (you planted them in Lab 02). **Why profile first:** the SK-01 discipline unchanged — measure, then fix, then prove. Write these numbers down; Step 4's reconciliation must account for every one of them.

### Step 3 — Clean, dedupe, quarantine

**What:**

```python
from pyspark.sql.window import Window

# 1. Deduplicate: keep one row per business key (deterministic pick)
w = Window.partitionBy("order_id", "product_id", "order_ts").orderBy(F.col("unit_price").desc_nulls_last())
deduped = (orders
    .withColumn("_rn", F.row_number().over(w))
    .filter("_rn = 1")
    .drop("_rn"))

# 2. Classify: valid vs quarantine (classify ONCE, route everywhere)
classified = deduped.withColumn(
    "reject_reason",
    F.when(F.col("quantity") <= 0, "non_positive_quantity")
     .when(F.col("unit_price").isNull(), "null_unit_price")
     .otherwise(F.lit(None))
)

valid = classified.filter("reject_reason IS NULL").drop("reject_reason")
rejects = classified.filter("reject_reason IS NOT NULL") \
                    .withColumn("quarantined_at", F.current_timestamp()) \
                    .withColumn("run_date", F.lit(run_date))

# 3. Derive and conform
silver_orders = (valid
    .withColumn("line_amount", F.round(F.col("quantity") * F.col("unit_price"), 2))
    .withColumn("order_date", F.to_date("order_ts")))
```

**Why each choice:**
- **Window + row_number** picks *one deterministic survivor* per duplicate group (ordering rule documented in code). `dropDuplicates()` picks arbitrarily — fine for identical rows, dangerous when duplicates differ; the window version is the professional default.
- **Classify-once with `when/otherwise`** — the SK-02 CASE pattern in Spark: every row gets exactly one verdict, and both routes come from the same computation (no drift between "what we kept" and "what we rejected").
- Rejects carry **reason + timestamp + run_date** — auditable, monitorable, reprocessable.

### Step 4 — Write silver idempotently (MERGE) and reconcile

**What:**

```python
# First run: create the tables. (MERGE needs an existing target.)
if not spark.catalog.tableExists("freshcart_silver.orders"):
    silver_orders.write.format("delta").saveAsTable("freshcart_silver.orders")
    rejects.write.format("delta").saveAsTable("freshcart_silver.orders_quarantine")
else:
    from delta.tables import DeltaTable
    target = DeltaTable.forName(spark, "freshcart_silver.orders")
    (target.alias("t")
        .merge(silver_orders.alias("u"),
               "t.order_id = u.order_id AND t.product_id = u.product_id AND t.order_ts = u.order_ts")
        .whenMatchedUpdateAll()
        .whenNotMatchedInsertAll()
        .execute())
    rejects.write.format("delta").mode("append").saveAsTable("freshcart_silver.orders_quarantine")

# RECONCILIATION: bronze = silver + quarantine (per this batch)
n_bronze  = deduped.count()
n_silver  = silver_orders.count()
n_reject  = rejects.count()
print(f"bronze(deduped)={n_bronze}  silver={n_silver}  quarantine={n_reject}")
assert n_bronze == n_silver + n_reject, "RECONCILIATION FAILED — rows unaccounted for"
```

**Why the assert is the star of the show:** it is the read = loaded + rejected identity, enforced *inside the job*. When Lab 05 schedules this notebook, a reconciliation failure fails the pipeline run and pages a human — bad math can never quietly publish.

**Prove idempotency (the ritual):** run the whole notebook again, top to bottom. Then:

```python
spark.sql("SELECT COUNT(*) FROM freshcart_silver.orders").show()
```

**Expected:** identical count to the first run (MERGE matched everything, inserted nothing). Note the quarantine *does* append per run — scope its dedup by `run_date` in the project if you rerun the same date, and mention this trade-off in your docs. Spotting that nuance is exactly the kind of thing reviewers probe.

**Troubleshooting:** `DeltaTable.forName` import errors → run on the default Fabric runtime (delta is preinstalled); concurrent-write conflicts → you left a second notebook session running; stop it in Monitoring hub.

### Step 5 — Build gold: the star schema

**What:** New notebook `02_silver_to_gold`, attach silver + gold (gold as default):

```python
from pyspark.sql import functions as F

orders   = spark.read.table("freshcart_silver.orders")
products = spark.read.table("freshcart_bronze.products_raw")   # reference data was clean
stores   = spark.read.table("freshcart_bronze.stores_raw")

# Dimensions (surrogate keys via hashing for stability across runs)
dim_product = (products
    .withColumn("product_key", F.xxhash64("product_id"))
    .select("product_key", "product_id", "category", "unit_cost"))

dim_store = (stores
    .withColumn("store_key", F.xxhash64("store_id"))
    .select("store_key", "store_id", "city", "opened_date"))

dim_date = (orders.select(F.col("order_date").alias("date")).distinct()
    .withColumn("date_key", F.date_format("date", "yyyyMMdd").cast("int"))
    .withColumn("year",  F.year("date"))
    .withColumn("month", F.month("date"))
    .withColumn("day_name", F.date_format("date", "EEEE")))

# Fact
fct_sales = (orders
    .withColumn("product_key", F.xxhash64("product_id"))
    .withColumn("store_key",   F.xxhash64("store_id"))
    .withColumn("date_key",    F.date_format("order_date", "yyyyMMdd").cast("int"))
    .select("date_key", "store_key", "product_key",
            "order_id", "quantity", "unit_price", "line_amount"))

for name, df in [("dim_product", dim_product), ("dim_store", dim_store),
                 ("dim_date", dim_date), ("fct_sales", fct_sales)]:
    df.write.format("delta").mode("overwrite").saveAsTable(f"freshcart_gold.{name}")
    print(name, spark.read.table(f"freshcart_gold.{name}").count())

# Gate: no orphan keys (referential integrity, SK-02 style)
orphans = (spark.read.table("freshcart_gold.fct_sales").alias("f")
    .join(spark.read.table("freshcart_gold.dim_product").alias("p"), "product_key", "left_anti"))
assert orphans.count() == 0, "GOLD GATE FAILED: fact rows with no product dimension"
```

**Why a star schema in a lakehouse:** because Lab 06's semantic model (and Power BI's Direct Lake mode) performs and behaves best over exactly this shape — dimensional modeling (SK-02) didn't die in the lake era; it moved in. **Why `mode("overwrite")` for gold:** gold is fully derivable from silver — rebuild-from-upstream beats incremental cleverness until scale forces otherwise (document this decision).

**Verify:** four tables in `freshcart_gold` with sane counts; the orphan gate passes.

### Step 6 — Housekeeping: history, time travel, OPTIMIZE

**What:**

```python
spark.sql("DESCRIBE HISTORY freshcart_silver.orders").select(
    "version", "timestamp", "operation", "operationMetrics.numTargetRowsInserted").show(truncate=False)

# Time travel: the table as it was before your second (idempotent) run
v0 = spark.read.option("versionAsOf", 0).table("freshcart_silver.orders")
print("Version 0 count:", v0.count())

spark.sql("OPTIMIZE freshcart_silver.orders")
```

**Why:** `DESCRIBE HISTORY` is your load-audit table, generated free by Delta (compare to the one you hand-built in SK-02). Time travel is the debugging superpower ("what did the table look like before last night's run?"). OPTIMIZE after MERGE-heavy work keeps file sizes healthy — and its output tells you how many files it compacted (evidence for your project's performance notes).

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Parameter cell as the pipeline seam** | Notebooks become schedulable, backfillable jobs, not interactive scratchpads. |
| **Profile → classify-once → quarantine with reasons** | The dead-letter discipline, in Spark; nothing silently dropped. |
| **Deterministic dedup (window + documented ordering)** | Survivor selection is a decision, not an accident. |
| **MERGE for silver** | Idempotency by construction; the run-twice ritual proves it. |
| **In-job reconciliation asserts (bronze = silver + quarantine; no orphans)** | Quality gates that fail the run — and, via Lab 05, alert humans. |
| **Gold as rebuildable star schema** | Dimensional modeling serves BI; rebuild-from-silver keeps it simple until scale demands more. |
| **DESCRIBE HISTORY / time travel / OPTIMIZE** | Free audit trail, debuggability, and file hygiene — Delta operational literacy. |

---

## 6. Reflection

### What you learned
Production-shaped Fabric notebooks: parameterized, profiled, classify-once cleaned, quarantined, MERGE-idempotent, reconciliation-gated — refining bronze through silver into a gold star schema, with Delta's history/time-travel/OPTIMIZE toolkit.

### Why it matters
This notebook pair is the beating heart of the FreshCart platform: Lab 05 schedules it, Lab 04's warehouse and Lab 06's dashboard consume its output, and the capstone extends it. It is also the single most interview-relevant artifact in this module — medallion + MERGE + quality gates is the modern lakehouse engineer's core loop.

### Interview questions (with model answers)

1. **"Explain the medallion architecture and why layers matter."**
   Bronze: immutable as-landed evidence. Silver: cleaned, deduplicated, conformed atoms with a quarantine for rejects. Gold: dimensional/business-ready. Layers separate concerns, enable rebuild-from-upstream, and stop every consumer re-implementing (and diverging on) the cleaning.

2. **"How do you make a Spark load idempotent?"**
   Delta MERGE on the business key (rerun = matched updates to identical values, no growth); or partition-scoped overwrite for full snapshots. Prove with run-twice + count/aggregate comparison.

3. **"How do you handle bad records in a notebook pipeline?"**
   Classify once (single when/otherwise), route failures to a quarantine Delta table with reason/timestamp/run_date, assert the reconciliation identity (in = valid + quarantined), and monitor quarantine growth.

4. **"dropDuplicates vs window-based dedup?"**
   dropDuplicates keeps an arbitrary row — fine only when duplicates are byte-identical. Window + row_number with an explicit ORDER BY picks a *documented, deterministic* survivor — required when duplicates differ.

5. **"What do DESCRIBE HISTORY and time travel give you operationally?"**
   A free per-write audit log (operation, metrics, timestamps) and the ability to query pre-incident versions — root-cause analysis and rollback material that file-based pipelines need custom machinery for.

6. **"Why does a lakehouse still need a star schema?"**
   BI engines (semantic models, Direct Lake) are optimized for facts/dimensions; conformed dimensions give consistent slicing; and the modeling clarifies grain and additivity. The lake changed the storage, not the consumption physics.

### Common interview traps
- "Silver means clean" without a quarantine story — where the bad rows *went* is the senior half of the answer.
- MERGE without knowing what makes it idempotent (matched updates are no-ops on identical data).
- Ignoring small-file compaction — mention OPTIMIZE when MERGE-heavy writes come up.

### Key takeaways
1. Bronze is evidence; silver is trust plus a quarantine; gold is a star.
2. MERGE + run-twice proof = lakehouse idempotency.
3. Asserts inside the notebook turn quality into a run-failing, human-alerting gate.
4. Delta gives you the audit table, time machine, and compactor — use all three.

**Next:** [Lab 04 — Data Warehouse and SQL Analytics](Lab_04_Data_Warehouse_and_SQL_Analytics.md) (already in your Labs folder), then [Lab 05](Lab_05_Pipelines_Orchestration_and_Monitoring.md) schedules everything you just built.
