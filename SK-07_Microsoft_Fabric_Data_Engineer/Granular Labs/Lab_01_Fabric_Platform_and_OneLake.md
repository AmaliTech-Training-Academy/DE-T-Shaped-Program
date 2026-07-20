# Lab 01 — The Fabric Platform and OneLake

> **Module:** SK-07 Microsoft Fabric Data Engineer
> **Time estimate:** 2.5–3.5 hours
> **Prerequisites:** Lab_00 completed (active trial, workspace `sk07-fabric-training`, folder `C:\FabricLabs\data`)
> **What you'll have at the end:** a working mental model of OneLake and Delta Parquet, OneLake browsable from Windows File Explorer, a working **shortcut**, and confident answers to "why Fabric instead of Databricks/Snowflake?"

---

## What You Will Build

Lab_00 got you *into* Fabric. This lab is about understanding the ground you're standing on before we start building FreshCart's platform in Lab_02. Concretely, you will:

1. Map each Fabric **experience** (Data Engineering, Data Factory, Data Warehouse, Real-Time Intelligence, Power BI) to the data-engineering job it does.
2. Understand **OneLake** — Fabric's single tenant-wide data lake — including its path structure and the "one copy of data" principle.
3. Understand **Delta Lake on Parquet**, the open table format everything in Fabric reads and writes.
4. Install the **OneLake file explorer** so OneLake appears inside Windows File Explorer like OneDrive does.
5. Create and test a **shortcut** — OneLake's zero-copy way of making data appear in a second place.
6. Compare Fabric honestly against Databricks, Snowflake, and a hand-assembled Azure stack.

This is the most reading-heavy lab in the module. That's deliberate: Labs 02–06 are hands-on-heavy, and they go much faster when these concepts are solid.

---

## 1. Environment Setup

### 1.1 Verify prerequisites from Lab_00

| # | Check | How to verify | Built in |
|---|---|---|---|
| 1 | Trial active | Profile picture (top right) → shows **Trial: N days left** | Lab_00 |
| 2 | Workspace exists | Left nav → **Workspaces** → `sk07-fabric-training` listed, License = Trial | Lab_00 |
| 3 | Local folder | PowerShell: `Test-Path C:\FabricLabs\data` returns `True` | Lab_00 |

If any check fails, return to Lab_00 — its Troubleshooting section covers every known failure mode.

### 1.2 What's new in this lab: the OneLake file explorer (Windows only)

The **OneLake file explorer** is a small free Windows app that mounts OneLake into Windows File Explorer, exactly the way OneDrive mounts your cloud files. It is optional but strongly recommended — seeing your cloud data as ordinary-looking folders demystifies OneLake faster than any diagram.

- **Install:** search the Microsoft Learn docs for "OneLake file explorer" and download the installer (an `.exe`; latest stable — the docs page always links the current build), or install from the Microsoft Store if listed. Standard next-next-finish installer; sign in with your **work/school** account when prompted.
- **Verify:** open **File Explorer** (`Win+E`). In the left navigation you should see a new node named **OneLake - Microsoft Fabric**. Expanding it shows your workspaces (it may be empty until Lab_02 creates items — that's fine).
- **Common install problems:**

| Problem | Fix |
|---|---|
| Node doesn't appear after install | Restart File Explorer: Task Manager → find **Windows Explorer** → **Restart** |
| Signed in with the wrong account | System tray → OneLake icon → Account → sign out, sign in with the training account |
| Corporate policy blocks the installer | Skip it — everything in this lab also works through the browser (`app.fabric.microsoft.com` → **OneLake** in the left nav) |

- **macOS/Linux:** the file explorer app is Windows-only. Use the **OneLake** hub in the browser's left navigation instead; every step below notes the browser path.

No other installs. Everything else happens in the browser.

---

## 2. Business Context

### FreshCart's actual problem is copies

Recall FreshCart from Lab_00: an online grocery startup whose "analytics" is a manager exporting CSVs into spreadsheets. Look closer at what's really wrong: the same order data exists in **five places** — the ops manager's export, the finance analyst's cleaned copy, a marketing spreadsheet, an old Access database, and the website's own database. Five copies, five versions of the truth, and nobody can say which is current. When the CEO asks "what was revenue last week?", she gets three different answers.

This is not a FreshCart quirk. In most companies, data engineering time is consumed not by *transforming* data but by **moving and synchronizing copies of it** — lake to warehouse, warehouse to BI extract, BI extract to someone's desktop. Every copy is a job that can fail, a security surface, a freshness lag, and a reconciliation argument waiting to happen.

**OneLake is Fabric's answer:** one logical lake for the whole organization, one copy of each dataset in an open format, and every engine — Spark, T-SQL, Power BI — reading that *same* copy. When you build FreshCart's platform in Labs 02–06, the gold tables your notebook writes are, byte-for-byte, the tables the warehouse queries and the Power BI report renders. No copy jobs. That is the single most important architectural fact in this module.

### Who consumes this, and what if it fails?

The consumers of a well-organized lake are *every* downstream lab: pipelines (Lab_02/05), notebooks (Lab_03), the warehouse (Lab_04), reports (Lab_06) — and every colleague who inherits your workspace. If the lake layer is chaotic (mystery folders, duplicated datasets, no naming standard), everything above it inherits the chaos: pipelines point at the wrong copy, reports disagree, and eventually the business stops trusting the platform — the most expensive failure in analytics.

---

## 3. Concept Explanation

### 3.1 The experiences — one product, several lenses

You toured the **experience switcher** in Lab_00. Now attach jobs to the names:

| Experience | Data-engineering job | Key items | You use it in |
|---|---|---|---|
| **Data Engineering** | Store and transform data at scale with Spark | Lakehouse, Notebook, Spark Job Definition | Labs 02, 03 |
| **Data Factory** | Move data and orchestrate work | Data pipeline, Dataflow Gen2 | Labs 02, 05 |
| **Data Warehouse** | Serve data to SQL users | Warehouse | Lab 04 |
| **Real-Time Intelligence** | Streaming/event data | Eventstream, Eventhouse | (out of scope for SK-07) |
| **Power BI** | Reports and dashboards | Semantic model, Report | Lab 06 |

Remember from Lab_00: switching experiences changes *which creation options are emphasized*, nothing more. All items live in workspaces; all data lives in OneLake.

### 3.2 OneLake — "OneDrive for data," taken literally

**What it is:** OneLake is a single, logical data lake automatically provisioned for your entire tenant (organization). You don't create it, size it, or configure it. Every workspace gets a folder inside it; every data item (lakehouse, warehouse...) gets a folder inside its workspace's folder. Under the hood it is built on Azure Data Lake Storage Gen2 — but as a SaaS user you never see or manage that.

**The path structure** is worth memorizing, because tools address data by it:

```
OneLake                                  ← one per tenant
└── sk07-fabric-training                 ← workspace
    └── freshcart_bronze.Lakehouse       ← item (you create this in Lab_02)
        ├── Files/                       ← free-form file area (any format)
        │   └── landing/orders_2026-07-06.csv
        └── Tables/                      ← managed Delta tables
            └── bronze_orders/
                ├── _delta_log/          ← Delta transaction log
                └── part-0000...parquet  ← the actual data
```

There is even a URI form (`abfss://sk07-fabric-training@onelake.dfs.fabric.microsoft.com/freshcart_bronze.Lakehouse/Tables/bronze_orders`) that Spark and external tools use — you'll meet it in Lab_03. **Files vs Tables** matters everywhere: *Files* is a folder for anything (raw CSVs, JSON, images); *Tables* is exclusively for Delta tables, which every Fabric engine automatically discovers and can query.

**Why "one copy" is a big deal — the tenant-wide claim:** OneLake spans workspaces. The finance team's workspace and the marketing team's workspace write into the *same* logical lake, with security controlling who sees what. Compare that to the common reality of one storage account (or S3 bucket) per team, each a silo needing credentials and copy jobs to cross.

### 3.3 Delta Lake — the format that makes "one copy" possible

Every table in Fabric — lakehouse or warehouse — is stored as **Delta Lake**: an open-source table format that takes **Parquet** files (a compressed, columnar file format that's the industry standard for analytics data) and adds a **transaction log** (`_delta_log/` — a folder of JSON files recording every change as a numbered, atomic commit). The log gives plain files database superpowers:

- **ACID transactions** — a half-finished write is never visible; readers see the last committed version.
- **Versioning / time travel** — the log remembers previous versions; you can query the table "as of" yesterday.
- **Schema enforcement** — writes with wrong columns are rejected instead of silently corrupting the table.

**Why this matters strategically:** because Delta is *open* (not a proprietary Microsoft format), the Spark engine, the T-SQL engine, and Power BI's Direct Lake mode can all read the same files — and so can external tools like Databricks. Fabric's whole "one copy" architecture stands on this choice. Alternatives in the market: **Apache Iceberg** and **Apache Hudi** are competing open table formats solving the same problem (Snowflake champions Iceberg); Fabric standardizes on Delta. Knowing all three names is interview gold.

### 3.4 Shortcuts — data in two places, stored in one

A **shortcut** is a OneLake object that makes data stored in one location *appear* inside another, with **zero copying** — like a Windows shortcut or a Unix symbolic link, but for lake data. Targets can be another OneLake location (another lakehouse, another workspace) or **external storage**: Amazon S3, Google Cloud Storage, or Azure Data Lake Gen2.

Why they exist: the alternative to a shortcut is a copy pipeline — which means duplication, lag, cost, and drift. With a shortcut, the marketing workspace can "see" the sales lakehouse's gold tables live, and a Fabric lakehouse can query data still sitting in a legacy S3 bucket without migrating it.

Two rules to burn in now (they reappear in Lab_04 and the capstone):
1. **Shortcuts are read-only through the shortcut.** You can never "fix" data through one — a feature, because it makes ownership unambiguous.
2. **A shortcut inherits the freshness of its target.** There is no sync job to break — but also no snapshotting; if the target changes, the shortcut sees it instantly.

### 3.5 Honest comparison — Fabric vs the alternatives

| | **Microsoft Fabric** | **Databricks** | **Snowflake** | **Raw Azure stack** (ADF+ADLS+Synapse+PBI) |
|---|---|---|---|---|
| Model | SaaS, all-in-one | PaaS, Spark-centric lakehouse | SaaS warehouse-centric | PaaS building blocks |
| Storage format | Delta (open) | Delta (open — they created it) | Proprietary + Iceberg support | Your choice |
| Strengths | Time-to-value, Power BI native, one bill, low ops | Best-in-class Spark, ML, fine control | Excellent SQL warehouse, sharing, multi-cloud | Maximum control |
| Weaknesses | Fewer knobs, newer/fast-changing | Assembly + tuning effort, BI not native | Orchestration/BI need partners; compute cost discipline | Integration burden is on you |
| Fits when… | Small team, M365 shop, BI-heavy | Heavy Spark/ML engineering org | SQL-first analytics org | Unusual requirements needing control |

FreshCart (tiny team, Microsoft shop, BI-first) → Fabric. A hedge fund with 40 Spark engineers → probably Databricks. *Fit the tool to the team* — you'll defend this reasoning in interviews (Section 6) and in the capstone README.

---

## 4. Step-by-Step Implementation

Nothing you do here can break anything, and everything gets cleaned up at the end.

### Step 1 — Create a scratch lakehouse to explore with

- **What:** In `sk07-fabric-training` → **+ New item** → **Lakehouse** → name it `onelake_explorer_demo` → **Create**. (A lakehouse is a storage item combining the Files and Tables areas from §3.2 — Lab_02 covers it fully; today it's our OneLake microscope.)
- **Why:** OneLake paths only become visible once at least one item exists to create them.
- **Expected output:** the lakehouse view opens: **Explorer** pane on the left showing **Tables** (empty) and **Files** (empty).
- **Verify:** the workspace item list now shows `onelake_explorer_demo` (type *Lakehouse*) plus an auto-created *SQL analytics endpoint* and *semantic model* of the same name — Fabric always creates these companions; you'll use them in Labs 04 and 06.
- **Common mistake:** creating it in **My workspace** instead of `sk07-fabric-training` — check the workspace name in the header first.

### Step 2 — Upload a file and load it to a Delta table

- **What:** In the Explorer pane: **Files** → `...` → **Upload → Upload files** → choose `C:\FabricLabs\data\setup_test.csv` (from Lab_00). Then click the uploaded file's `...` → **Load to Tables → New table** → name `setup_test` → **Load**.
- **Why:** this one action shows the Files-vs-Tables boundary: the CSV sits in Files as-is; "Load to Tables" converts it into a **Delta** table.
- **Expected output:** after ~30 seconds, `setup_test` appears under **Tables** with a triangle/table icon. Clicking it previews 2 rows.
- **Verify (the interesting part):** under **Tables**, click `setup_test` → `...` → **View files** (or open the table folder). You'll see `_delta_log/` and one or more `part-*.parquet` files — §3.3 made real.
- **Troubleshooting:** *Load to Tables greyed out* → wait for upload to finish; *table lands under "Unidentified"* → the CSV was malformed — regenerate it with Lab_00 §1.5 Step 6.

### Step 3 — See the same data through Windows File Explorer

- **What (Windows):** open File Explorer → **OneLake - Microsoft Fabric** → `sk07-fabric-training` → `onelake_explorer_demo.Lakehouse` → browse `Files\` and `Tables\setup_test\`.
- **What (browser/macOS):** left nav → **OneLake** (or **Browse → OneLake data hub**) → locate the lakehouse → browse its folders.
- **Why:** proof that OneLake is just a governed file system — the "OneDrive for data" line becoming literal. The parquet files you see are the *actual* single copy of the table.
- **Expected output:** the same `part-*.parquet` and `_delta_log` you saw in the browser.
- **Common mistake:** editing or deleting files under `Tables\` from File Explorer. **Never do this** — hand-editing a Delta table's files corrupts its transaction log (the "Unidentified" tables warned about in Lab_06 are usually exactly this). Treat File Explorer access as read-only for Tables; only engines that speak Delta should write there.
- **Troubleshooting:** empty node → system-tray OneLake icon → **Sync** / sign out-in; still empty → use the browser path, functionality is identical.

### Step 4 — Create a shortcut and prove it's zero-copy

- **What:**
  1. Create a second lakehouse: **+ New item → Lakehouse** → `shortcut_target_demo`.
  2. Inside it: **Tables** → `...` → **New shortcut** → **Microsoft OneLake** → select `onelake_explorer_demo` → tick table `setup_test` → **Create**.
- **Why:** this simulates the everyday pattern "team B needs team A's table without copying it."
- **Expected output:** `setup_test` appears under `shortcut_target_demo` → Tables, with a small **link badge** on its icon. Preview shows the same 2 rows.
- **Verify zero-copy:** go back to `onelake_explorer_demo`, upload a second small CSV row set to the *original* table? Simpler: rename nothing, instead note the shortcut has **no files of its own** — in the OneLake file explorer, `shortcut_target_demo.Lakehouse\Tables\setup_test` resolves to the same underlying data (no duplicate parquet files consuming storage; the OneLake data hub shows no added storage for the second lakehouse).
- **Verify read-only:** shortcuts offer no upload/write options through the shortcut side — confirm the `...` menu on the shortcut table lacks load/write actions. Rule 1 from §3.4, observed.
- **Troubleshooting:** *shortcut creation fails with a permissions error* → you must have access to the target item (you do — you own both; in real teams this is where workspace roles matter). *Table not listed in the picker* → the target table isn't a valid Delta table; redo Step 2.

### Step 5 — Clean up

- **What:** in the workspace item list, delete `shortcut_target_demo` first, then `onelake_explorer_demo` (hover → `...` → **Delete**), confirming each. Deleting a shortcut **never deletes the target data** — but deleting the *target* leaves any remaining shortcuts dangling, which is why we delete in this order.
- **Verify:** workspace item list is empty again — the Lab_00 "no junk drawer" habit.
- **Why it matters:** in the capstone, a dangling shortcut is a rubric deduction; in production, it's a 2 a.m. page.

---

## 5. Production Engineering Practices

**Zones and layout are governance.** Real OneLake estates agree a layout *before* building: one workspace per team-environment (`freshcart-prod`), one lakehouse per medallion layer, raw files under `Files/landing/<source>/<date>/`. You'll follow exactly this in Lab_02. *Failure story:* a retailer let every analyst create lakehouses freely; 18 months later, 340 lakehouses, 60 of them named some variant of "test", and a compliance audit (GDPR data-deletion request) took three weeks because nobody could say where customer data lived. Layout discipline is not tidiness — it's the ability to answer "where is X?" under legal deadline.

**Shortcuts as contracts.** When team B shortcuts to team A's table, A's schema becomes a **public API**: renaming a column breaks B silently. Production teams document who consumes what (Lab_06 develops this into the "semantic contract" idea) and announce schema changes like API changes.

**Don't touch Tables/ by hand.** You saw why in Step 3. The production rule: humans browse, engines write. Anything under `Tables/` is owned by whatever pipeline writes it — fixes go through the pipeline (idempotent re-runs, Lab_05), never through File Explorer.

**Open formats are an exit strategy.** Choosing Delta means FreshCart's data is never hostage to a vendor: the same files are readable by Databricks or open-source Spark tomorrow. When you justify platform choices to a CTO (capstone `01_Business_Scenario.md` has one), "our data stays in an open format" is a first-class argument.

---

## 6. Reflection

### What you learned

- Fabric's experiences are lenses on one product; OneLake is the single tenant-wide lake beneath all of them, with a predictable `workspace/item/Files|Tables` path structure.
- Every table is **Delta Lake**: Parquet files plus a transaction log giving ACID, versioning, and schema enforcement — the open format that makes "one copy, many engines" possible.
- **Shortcuts** expose data in a second place with zero copies, read-only, always-fresh.
- Hands-on: mounted OneLake in Windows File Explorer, loaded a CSV to a Delta table, inspected its parquet + `_delta_log` anatomy, and built and destroyed a shortcut safely.

### Why it matters

Every subsequent lab manipulates Delta tables in OneLake — Lab_02 lands data into it, Lab_03 transforms within it, Labs 04/06 serve straight out of it. Engineers who understand the storage layer debug in minutes what mystifies others for days ("why is my table Unidentified?" — you already know).

### Interview questions

**Q1. "What is OneLake?"**
*Model answer:* Fabric's single, logical, tenant-wide data lake, automatically provisioned, built on ADLS Gen2 but fully managed. Every workspace and item maps to a folder in it; all engines read/write the same copy of data in open Delta format. Its pitch is eliminating per-team storage silos and copy pipelines.
*Trap:* calling it "a storage account you create" — you never create or configure OneLake.

**Q2. "What is Delta Lake and why does Fabric use it?"**
*Model answer:* An open table format layering a transaction log over Parquet files, adding ACID transactions, versioning/time travel, and schema enforcement. Fabric standardizes on it so Spark, T-SQL, and Power BI Direct Lake can all read one copy of the data — and so customers aren't locked into a proprietary format.
*Trap:* saying Delta is a *file* format — Parquet is the file format; Delta is the *table* format around it. Bonus: name Iceberg and Hudi as alternatives.

**Q3. "What is a OneLake shortcut and when would you use one instead of copying?"**
*Model answer:* A symbolic-link-like object making data in another OneLake location or external store (S3, GCS, ADLS) appear locally with zero copies — always fresh, read-only through the shortcut. Use it for cross-team/cross-workspace sharing and for querying data in legacy storage without migration. Copy instead when you need a point-in-time snapshot or must transform on the way in.
*Trap:* claiming shortcuts sync or cache data — there is nothing to sync; it's the same bytes.

**Q4. "Files vs Tables in a lakehouse?"**
*Model answer:* Files is unmanaged storage for any format — landing zones, raw CSVs. Tables holds only Delta tables, automatically discovered and queryable by every Fabric engine, including the SQL analytics endpoint. Ingestion typically lands raw data in Files, then loads/transforms it into Tables.
*Trap:* trying to explain SQL queries over raw CSVs in Files — the SQL endpoint only sees Tables.

**Q5. "Your company is Microsoft-centric with a small data team. A vendor pitches Databricks. What do you weigh?"**
*Model answer:* Databricks wins on Spark depth, ML tooling, and control knobs; Fabric wins on time-to-value, native Power BI, one bill, and lower ops for a small team. Both use open Delta, so data is portable either way. I'd match the tool to the team's skills and the workload's actual needs, and pilot before committing.
*Trap:* an absolute answer either way — the interviewer wants trade-off reasoning.

**Q6. "Can Fabric read data that lives in Amazon S3?"**
*Model answer:* Yes — an S3 shortcut exposes the bucket's data in OneLake without migration; Fabric engines query it in place (egress latency/cost considerations aside). This is a common bridge strategy during multi-cloud migrations.
*Trap:* assuming everything must be copied into OneLake first.

### Key takeaways

1. **One copy, many engines** — Fabric's core promise, made possible by Delta in OneLake.
2. **The lake is just governed files** — you browsed it in File Explorer; nothing is magic, and `_delta_log` is the magician.
3. **Never hand-edit Tables/**; shortcuts are read-only; layouts are governance.

### Next lab

➡️ **Lab_02_Lakehouse_and_Data_Ingestion.md** — you create `freshcart_bronze`, generate FreshCart's sample dataset (18 months of orders), and ingest it three ways: manual upload, a pipeline **Copy activity**, and a **Dataflow Gen2**.
