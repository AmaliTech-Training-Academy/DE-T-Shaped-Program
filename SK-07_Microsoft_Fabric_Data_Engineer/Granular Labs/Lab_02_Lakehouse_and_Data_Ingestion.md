# Lab 02 — Lakehouse and Data Ingestion

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Estimated time:** 3–4 hours
> **Prerequisites:** Lab 01 — you have a Fabric trial capacity, the `FreshCart-Dev` workspace, and you understand OneLake/items/capacities.

---

## 1. Environment Setup

### 1.1 Verify your Fabric state

1. Sign in at **app.fabric.microsoft.com**.
2. Open your `FreshCart-Dev` workspace — confirm the capacity badge (trial diamond icon) is present. No capacity = items won't create.
3. Check remaining trial days (Account manager → Trial status). Plan your module pace around it.

### 1.2 Generate the FreshCart source data

We simulate the exports FreshCart's systems drop each night. Create `C:\FabricLabs\data` and run this locally (any Python 3.11+; no extra packages beyond pandas):

```python
# make_freshcart_data.py — creates 3 realistic, imperfect CSVs
from pathlib import Path
import pandas as pd, random
random.seed(42)
out = Path(r"C:\FabricLabs\data"); out.mkdir(parents=True, exist_ok=True)

stores = [f"S{i:02d}" for i in range(1, 11)]
products = [(f"P{i:03d}", random.choice(["Produce","Dairy","Bakery","Frozen","Pantry"]),
             round(random.uniform(0.5, 25), 2)) for i in range(1, 201)]

rows = []
for day in pd.date_range("2026-05-01", "2026-06-30"):
    for _ in range(random.randint(300, 500)):
        pid, cat, price = random.choice(products)
        qty = random.choice([1,1,1,2,2,3,4,5])
        rows.append({
            "order_id": f"O{random.randint(100000, 999999)}",
            "order_ts": (day + pd.Timedelta(minutes=random.randint(480, 1320))).isoformat(),
            "store_id": random.choice(stores),
            "product_id": pid,
            "quantity": qty if random.random() > 0.005 else -qty,       # ~0.5% bad rows
            "unit_price": price if random.random() > 0.003 else None,    # some nulls
        })
df = pd.DataFrame(rows)
dupes = df.sample(frac=0.01, random_state=1)                             # ~1% duplicates
pd.concat([df, dupes]).to_csv(out / "orders.csv", index=False)

pd.DataFrame([{"product_id": p, "category": c, "unit_cost": round(pr*0.6, 2)}
              for p, c, pr in products]).to_csv(out / "products.csv", index=False)
pd.DataFrame([{"store_id": s, "city": random.choice(["Accra","Kumasi","Takoradi","Tema"]),
               "opened_date": "2019-0%d-01" % random.randint(1, 9)}
              for s in stores]).to_csv(out / "stores.csv", index=False)
print("Created:", [f.name for f in out.glob("*.csv")])
```

**Verify:** three CSVs exist; `orders.csv` is ~25k+ rows. The deliberate defects (negative quantities, null prices, duplicates) are Lab 03's raw material — real bronze data is never clean, so ours isn't either.

### 1.3 Common problems

| Symptom | Fix |
|---|---|
| "Create" button greyed out in workspace | Workspace not assigned to capacity — Workspace settings → License info → Trial |
| Trial expired | Fabric admin can extend / start a new trial tenant; see Lab 00 §Troubleshooting |
| Upload stalls | Corporate VPN throttling — try without VPN or use OneLake file explorer (installed in Lab 00) |

---

## 2. Business Context

FreshCart is a grocery chain whose nightly exports (orders, products, stores) currently land in email attachments and a shared drive — "our data platform is a folder," as their ops lead puts it. Your job across Labs 02–06 is to build their platform on Fabric: land raw data reliably (**this lab**), refine it through medallion layers (Lab 03), serve SQL analytics (Lab 04), orchestrate nightly (Lab 05), and ship dashboards (Lab 06).

**Why a lakehouse?** FreshCart needs *both* worlds: file-flexibility for whatever lands (CSV today, JSON from the new POS next quarter) *and* table-semantics for SQL and BI. A **lakehouse** is exactly that hybrid — files in a data lake, exposed as ACID **Delta tables** queryable by Spark, SQL, and Power BI simultaneously.

**Who consumes today's output?** Lab 03's Spark notebooks read the bronze tables you land here. **If ingestion is unreliable**, everything downstream inherits the gaps — which is why we practice *three* ingestion methods and pick deliberately, and why row counts are verified at every landing.

---

## 3. Concept Explanation

### 3.1 The lakehouse item

Creating a Lakehouse in Fabric gives you two zones in one item:

- **Files/** — raw object storage (any format, any structure). Your landing area.
- **Tables/** — **Delta Lake** tables: Parquet files + a transaction log giving ACID guarantees, schema enforcement, and time travel. What Spark/SQL/Power BI query.

Everything physically lives in **OneLake** (Lab 01) — one logical lake for the whole tenant, so nothing is copied between engines.

### 3.2 Delta Lake in one paragraph

Delta = Parquet files + `_delta_log` (a JSON transaction log). The log makes writes atomic (readers never see half a write — the atomic-write pattern from earlier modules, built into the format), enables `MERGE` (upserts — idempotency's best friend, starring in Lab 03), schema enforcement (bad-shape writes rejected), and **time travel** (query the table as of yesterday). It is the default table format across Fabric, Databricks, and much of the modern lake world.

### 3.3 Three ways to ingest (and when each wins)

| Method | What it is | Best for | Watch out |
|---|---|---|---|
| **Manual upload** | Drag files into Files/ via browser or OneLake explorer | One-offs, exploration, this lab's first step | Not repeatable — never a production answer |
| **Pipeline Copy activity** | A data-factory-style activity moving data source→destination at scale | Scheduled bulk movement, many sources, orchestration (Lab 05 builds on this) | It moves data; it doesn't transform much |
| **Dataflow Gen2** | Power Query in the cloud — visual, step-based extract/transform/load | Analyst-friendly transformation during ingestion; Excel/CSV/API sources | Heavier per-row than Spark at big volumes |

Rule of thumb you'll apply in the project: **Copy activity to land raw fast; transformations later in Spark (engineering) or Dataflows (analyst self-service); manual only for experiments.**

---

## 4. Step-by-Step Implementation

### Step 1 — Create the bronze lakehouse

**What:** Workspace → **+ New item → Lakehouse** → name `freshcart_bronze`. Explore what appeared: the lakehouse item, plus an auto-created **SQL analytics endpoint** and **default semantic model** (you'll use both later).

**Why the name:** medallion layers as separate lakehouses (`freshcart_bronze/silver/gold`) gives clean security boundaries and obvious lineage. (Single-lakehouse-with-schemas is a valid alternative at small scale — know both; we choose separation for clarity.)

**Verify:** the lakehouse opens showing empty `Tables/` and `Files/` panes.

### Step 2 — Ingest method 1: manual upload

**What:** In `freshcart_bronze` → Files/ → **Upload → Upload files** → select your three CSVs. Create a folder first: `Files/landing/2026-06-30/` (the "one folder per drop date" convention).

**Why the dated folder:** it mirrors how real drops arrive and sets up Lab 05's parameterized loads ("process folder for date X") — the partition-scoped idempotency pattern from earlier modules.

**Verify:** three files listed with sizes. Click `orders.csv` → preview renders.

**Now convert a file to a Delta table** (upload put it in Files/, not Tables/): right-click `orders.csv` → **Load to Tables → New table** → name `orders_raw`. Fabric infers the schema and writes Delta.

**Expected output:** `orders_raw` appears under Tables/ with a triangle-logo (Delta) icon. Open it — note Fabric read the CSV and typed the columns (check `quantity` became bigint, `order_ts` may be string — *inference is convenience, not contract*; Lab 03 applies real schema discipline).

**Verify the count:** lakehouse toolbar → **SQL analytics endpoint** (top-right switcher) → run:

```sql
SELECT COUNT(*) AS row_count FROM orders_raw;
```

Compare to your local `orders.csv` line count (minus header). **Reconciliation at every landing** — the habit transfers unchanged from SK-01.

### Step 3 — Ingest method 2: pipeline Copy activity

**What:**
1. Workspace → **+ New item → Data pipeline** → name `pl_ingest_freshcart`.
2. **Copy data assistant** (or add a Copy activity manually): Source = the sample "HTTP" won't apply here — use **Local files? No:** pipelines can't reach your laptop. This is the moment to learn the boundary: cloud pipelines pull from *reachable* sources (cloud storage, databases, HTTP APIs, or on-prem via a gateway). So we simulate: Source = the `Files/landing/` area of `freshcart_bronze` itself (connector: Lakehouse), path `landing/2026-06-30/products.csv`, format DelimitedText with header.
3. Destination = same lakehouse, **Tables**, table name `products_raw`, action **Overwrite**.
4. Repeat (second Copy activity, chained on success) for `stores.csv` → `stores_raw`.
5. **Run** the pipeline (▶). Watch the Output pane: each activity turns `Succeeded`, with rows-read/rows-written counts.

**Why Overwrite:** re-running this pipeline must not duplicate reference data — overwrite-by-default for full snapshots is the simplest idempotency (facts will use smarter scoping in Labs 03/05).

**Why the counts in the Output pane matter:** that's the pipeline's built-in reconciliation — rows read = rows written, visible per run, kept in run history. You'll wire alerts to this in Lab 05.

**Verify:**

```sql
SELECT COUNT(*) FROM products_raw;   -- 200
SELECT COUNT(*) FROM stores_raw;     -- 10
```

**Common mistakes:** choosing "Append" for reference tables (duplicates on every rerun); forgetting "First row as header" (columns become `Prop_0, Prop_1...`).

### Step 4 — Ingest method 3: Dataflow Gen2

**What:**
1. Workspace → **+ New item → Dataflow Gen2** → name `df_orders_typed`.
2. **Get data → Lakehouse** (or Text/CSV pointing at the landing file) → select `orders.csv`.
3. In the Power Query editor, do *light* typing only (bronze stays raw-ish): set column types — `order_ts` → Date/Time, `quantity` → Whole number, `unit_price` → Decimal. Watch the **Applied steps** list record each action — this is a visual, auditable transformation script (the M language behind it is viewable via Advanced editor).
4. Set the **data destination** (bottom-right): `freshcart_bronze` → table `orders_typed`, Replace.
5. **Publish**, then watch the refresh complete (workspace item → Refresh history).

**Why show a third method for the same data:** because choosing ingestion tooling *is the job*. You now have direct experience for the comparison table in §3.3 — and you'll defend a choice in the capstone. Note what Dataflow gave you that Copy didn't (typed columns, step-by-step auditability) and what it cost (slower, more moving parts).

**Verify:**

```sql
SELECT TOP 5 * FROM orders_typed;    -- order_ts should be datetime2, not varchar
```

### Step 5 — Inspect Delta's machinery (5 minutes that demystify everything)

**What:** Files pane → open `Tables/orders_raw/` (toggle "show hidden" if needed): see the Parquet part-files and the `_delta_log/` folder; open a JSON log entry and skim it (add/remove file actions, schema, timestamps).

Then time-travel via the SQL endpoint (Spark syntax comes in Lab 03):

```sql
DESCRIBE HISTORY orders_raw;    -- run in a notebook in Lab 03; for now, note the Versions in the table's context menu
```

**Why:** Delta stops being magic once you've seen it's just Parquet + a JSON log. Atomicity, MERGE, and time travel all follow from "readers only trust what the log commits" — one idea, many superpowers.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Dated landing folders (`landing/YYYY-MM-DD/`)** | Sets up parameterized, re-runnable, partition-scoped loads (Lab 05). |
| **Reconcile counts at every landing** (SQL endpoint vs source) | Partial loads caught the minute they happen, not at month-end. |
| **Overwrite semantics for snapshot/reference loads** | Rerun-safe by construction — idempotency as the default choice, not an afterthought. |
| **Deliberate tool choice (upload vs Copy vs Dataflow)** | Architecture judgment, documented; manual upload explicitly banned from production paths. |
| **Bronze keeps data raw(ish); typing is light and logged** | The medallion contract: bronze = evidence, transformations belong downstream. |
| **Understanding Delta's log** | You can now reason about atomic writes, concurrent readers, and time travel instead of memorizing features. |

---

## 6. Reflection

### What you learned
Creating a lakehouse, landing FreshCart's (deliberately imperfect) data three different ways, converting files to Delta tables, reconciling counts, and seeing Delta's transaction log with your own eyes.

### Why it matters
Ingestion choices ripple through everything downstream. And the lakehouse + Delta combination you touched today is the architectural core of Fabric — Labs 03–06 all build directly on these bronze tables.

### Interview questions (with model answers)

1. **"What is a lakehouse and how does it differ from a warehouse and a lake?"**
   Lake: cheap file storage, any format, no table semantics. Warehouse: managed tables, SQL, ACID — but structured-only and load-first. Lakehouse: files in the lake exposed as ACID Delta tables — one copy of data serving Spark, SQL, and BI. Fabric's Lakehouse item implements exactly this on OneLake.

2. **"What does Delta Lake add over plain Parquet?"**
   A transaction log: atomic commits (no torn reads), MERGE/UPDATE/DELETE, schema enforcement, and time travel. Parquet is the storage; the log is the database-like behavior.

3. **"Copy activity vs Dataflow Gen2 — when each?"**
   Copy: fast bulk movement, many connectors, orchestration-friendly — landing raw data. Dataflow Gen2: Power Query transformations during ingestion, analyst-accessible, step-audited — lighter-volume shaping. Land fast with Copy, transform downstream (Spark for engineering scale, Dataflows for self-service).

4. **"How do you make ingestion re-runnable?"**
   Overwrite for full snapshots; partition/date-scoped overwrite or MERGE for incremental; dated landing folders so a rerun targets exactly one drop; and count reconciliation on every run to prove it.

5. **"Where does lakehouse data physically live in Fabric?"**
   OneLake — the tenant-wide logical lake (ADLS Gen2 under the hood). Every workspace/item maps to a OneLake path; engines share the one copy (no import copies between Spark, SQL, and Direct Lake BI).

6. **"Why keep bronze raw instead of cleaning on ingestion?"**
   Bronze is the audit trail and reprocessing basis — the immutable-raw principle. Clean in silver, where fixes are re-runnable against preserved originals.

### Common interview traps
- Calling Delta "a file format" — it's Parquet *plus a transaction log*; the log is the answer.
- "I'd upload the files" for a production scenario — manual paths are for exploration only; say pipeline + schedule + reconciliation.
- Not knowing the auto-created SQL endpoint/semantic model exist — Fabric interviewers expect the item model.

### Key takeaways
1. Lakehouse = Files/ for landing, Tables/ (Delta) for consumption — one item, both worlds.
2. Delta = Parquet + log ⇒ atomicity, MERGE, time travel.
3. Three ingestion tools, one decision framework: land fast, transform downstream, never manual in production.
4. Reconcile every landing; date every drop; overwrite snapshots.

**Next:** [Lab 03 — PySpark Notebooks and the Medallion Architecture](Lab_03_PySpark_Notebooks_and_Medallion_Architecture.md): the defects you planted today (negative quantities, nulls, duplicates) meet Spark, quarantine tables, and Delta MERGE.
