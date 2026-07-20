# Lab 01 — Power BI & BI Concepts: Your First (Deliberately Naive) Report

> **Module:** SK-06 Power BI Analytics Practitioner
> **Estimated time:** 4–5 hours
> **Builds on:** [Lab_00_Environment_Setup.md](Lab_00_Environment_Setup.md) — you need Power BI Desktop installed, Auto date/time disabled, and the NorthStar CSVs generated in `C:\PowerBI_Labs\NorthStar\data\`.
> **Produces:** `C:\PowerBI_Labs\NorthStar\pbix\NorthStar_Lab01.pbix`

In this lab you learn what Business Intelligence actually *is*, how the Power BI ecosystem fits together, and then you build your first report the way every beginner does — by dumping a flat file onto a canvas. It will *appear* to work, then break in three specific ways. Those breaks are the whole point: they motivate everything in Labs 02–05.

---

## 1. Environment Setup

Lab_00 did the heavy lifting. Verify it before continuing — nothing new is installed in this lab.

**Verification checklist (2 minutes):**

1. **Power BI Desktop opens.** Start menu → "Power BI Desktop". If it was installed via the Microsoft Store it auto-updates monthly; check the version via **Help → About** (any version from the last ~6 months is fine — the UI moves slowly).
2. **Auto date/time is OFF.** File → Options and settings → Options → **Global → Data Load** → "Auto date/time for new files" must be **unchecked**. If you skipped this in Lab_00, fix it now — everything in Labs 03/05 assumes it.
3. **Data exists.** In PowerShell:

   ```powershell
   Get-ChildItem C:\PowerBI_Labs\NorthStar\data\*.csv | Select-Object Name, Length
   ```

   Expected: six files (`stores`, `products`, `customers`, `promotions`, `sales`, `targets`), with `sales.csv` around 4–6 MB. Missing files? Re-run the Lab_00 generation script.
4. **Working folder for .pbix files exists:**

   ```powershell
   New-Item -ItemType Directory -Force C:\PowerBI_Labs\NorthStar\pbix | Out-Null
   ```

**Common problem:** Power BI Desktop won't start or crashes on launch on some VMs — usually a graphics driver issue. Fix: Options → Global → Report settings → check "Use software rendering" (requires restart).

macOS/Linux note: Power BI Desktop is Windows-only. Use a Windows VM as described in Lab_00; everything else in this lab is identical inside the VM.

---

## 2. Business Context

### The problem NorthStar has

Recall the scenario from the README: every Monday, a NorthStar analyst spends the day assembling a **40-page Excel report** for the CFO. Data is copied from exports, formulas are dragged down, charts are re-pasted. The report is:

- **Slow** — a full person-day per week, ~50 person-days a year, on copy-paste.
- **Error-prone** — one dragged formula off by a row and the CFO makes a decision on wrong numbers.
- **Stale** — the moment it's emailed, it's frozen. "What about just the West region?" means *another* day of work.
- **Untrusted** — when two spreadsheets disagree, meetings become arguments about whose number is right instead of what to do.

### Why companies need BI

**Business Intelligence (BI)** is the discipline of turning stored data into information people can act on: reports, dashboards, self-service exploration. Every mid-size-or-larger company has this problem, which is why BI developers and analytics engineers are among the most consistently in-demand data roles.

Who consumes BI output? Executives (KPI dashboards), managers (operational reports), analysts (self-service exploration), and increasingly customers themselves (embedded analytics — exactly what the CloudHR capstone does).

**What happens if it fails?** Decisions get made on stale or wrong numbers; teams keep private spreadsheet copies ("shadow BI") that drift apart; and trust, once lost, takes months to rebuild. In regulated industries, a wrong reported number can be a legal problem, not just an embarrassing one.

### Where this sits in *your* value chain

In SK-01 through SK-05 you built pipelines that land clean data in storage. BI is the **consumption layer** on top:

```mermaid
flowchart LR
    SRC["Source systems<br/>(POS, ERP, apps)"] --> ETL["Pipelines<br/>(SK-03/04/05)"]
    ETL --> WH["Warehouse / lake<br/>(SK-02/04)"]
    WH --> SM["Semantic model<br/>(THIS MODULE)"]
    SM --> RPT["Reports & dashboards"]
    RPT --> HUMANS["Decisions"]
```

The **semantic model** — a business-friendly layer of tables, relationships, and calculations — is the piece most engineers have never built. It's what Labs 02–05 teach.

---

## 3. Concept Explanation

### 3.1 The Power BI ecosystem (know these five, in interviews too)

| Component | What it is | Cost | You use it in this module? |
|-----------|-----------|------|---------------------------|
| **Power BI Desktop** | Windows authoring app: ingest, model, calculate, design | Free | Yes — the entire module |
| **Power BI Service** | Cloud portal (app.powerbi.com): sharing, scheduled refresh, workspaces, apps | Free tier exists; sharing needs Pro (~$14/user/mo) or capacity | Concepts only (Lab_06) |
| **Power BI Mobile** | Phone/tablet apps rendering mobile layouts | Free apps | Concepts only (mobile layout in Lab_06) |
| **Power BI Report Server** | On-premises report portal for orgs that can't use cloud | Paid | No — awareness only |
| **Microsoft Fabric** | Unified analytics platform; Power BI is one of its workloads | Capacity-based | SK-07 |

Mental model: **Desktop is the workshop, the Service is the gallery.** You build locally in a `.pbix` file; you publish to the Service so others can view. This module stays in the workshop; Lab_06 explains the gallery conceptually.

### 3.2 What's inside a .pbix file

A `.pbix` is a zip-like container holding three distinct things — and this separation is the most important architectural idea in Power BI:

1. **Queries (Power Query / M)** — instructions for getting and shaping data (Lab_02).
2. **The semantic model** — imported data (compressed, in-memory, columnar — the "VertiPaq" engine), relationships, and DAX measures (Labs 03–05).
3. **The report** — pages of visuals bound to the model (Lab_06).

Alternatives to Power BI: **Tableau** (strong visualization heritage, weaker built-in modeling language), **Looker** (semantic layer defined in code, BigQuery-centric), **Apache Superset/Metabase** (open source, lighter modeling), plain **Excel** (ubiquitous, but no shared model or governance). Power BI's advantages: bundled with Microsoft 365 (hence enormous adoption), a genuinely powerful modeling engine, and DAX. Its disadvantages: Windows-only authoring and a licensing model that takes a diagram to explain.

### 3.3 The three views in Desktop

Down the left edge of Power BI Desktop are three (sometimes four) icons:

- **Report view** — the canvas where visuals live.
- **Table view** — see the actual rows loaded into the model (like browsing a table in SSMS from SK-02).
- **Model view** — the diagram of tables and relationships (Lab_03's home).
- **DAX query view** (newer versions) — run DAX queries directly (used in Labs 04–05).

### 3.4 Import vs DirectQuery (one paragraph now, depth later)

When connecting to data, Power BI can **Import** (copy data into the in-memory model — fast, the default, refreshed on a schedule) or use **DirectQuery** (leave data at the source and translate every visual into a live query — always current, usually slower). This module uses **Import** throughout; you'll justify that choice again in the capstone (constraint C-04).

---

## 4. Step-by-Step Implementation

You will now do the **naive thing on purpose**: load only `sales.csv` flat, build visuals, and watch it fail. Professionals know *why* the right way is right because they've seen the wrong way break.

### Step 1 — Create the file and connect to sales.csv

1. Open Power BI Desktop → close the splash dialog → **File → Save As** → `C:\PowerBI_Labs\NorthStar\pbix\NorthStar_Lab01.pbix`.
   *Why save first?* Naming and versioning from minute one is a production habit (see Section 5). The file will be small until data is loaded.
2. **Home ribbon → Get data → Text/CSV** → navigate to `C:\PowerBI_Labs\NorthStar\data\sales.csv` → **Open**.
3. A preview window appears showing the first 200 rows and the detected delimiter/types.
   **Expected output:** columns like `order_id`, `order_date`, `store_id`, `product_id`, `customer_id`, `quantity`, `unit_price`, `discount_pct` and a promotion column, with `File Origin` UTF-8 and delimiter Comma.
4. Click **Load** (NOT "Transform Data" — this lab is the naive path; Lab_02 is where Transform Data becomes your default).

**Verification:** the Data pane on the right now shows a `sales` table. Switch to **Table view** and confirm rows are visible. In the bottom-left status bar nothing should say "errors".

**Common mistakes:**
- Picking **Transform Data** and getting "stuck" in a new window — that's the Power Query editor (Lab_02's home). Click **Close & Apply** to get back.
- Preview shows one giant column: the delimiter was misdetected. Set Delimiter = Comma in the preview dialog.

**Troubleshooting:** "Could not find file" after moving folders — data source paths are stored absolute. Home → Transform data → Data source settings → Change Source. (Lab_02 fixes this properly with parameters.)

### Step 2 — First visual: revenue over time

1. In **Report view**, check the Data pane. Notice `sales` has no "revenue" column — the CSV has `quantity`, `unit_price`, and `discount_pct`. For now, use quantity; revenue math comes in Lab_04 (and a taste in Step 4).
2. Click a blank canvas area → in the **Visualizations** pane click the **Line chart** icon.
3. Drag `order_date` to the **X-axis** well and `quantity` to the **Y-axis** well.

**Expected output:** a line chart spanning roughly two years of dates, quantity fluctuating with visible weekly rhythm.

**What just happened (important):** you never wrote `GROUP BY`, but Power BI summed `quantity` per date automatically. Numeric columns get a **default aggregation** (usually Sum) — that little `Σ` next to the field name. Every visual is implicitly a `SELECT date, SUM(quantity) ... GROUP BY date` against the model. Coming from SK-02, that's the fastest way to understand visuals.

**Common mistake:** if your X-axis shows date *hierarchies* (Year/Quarter/Month/Day), Auto date/time is still on — go back to Section 1 and disable it, then reload.

### Step 3 — Add a slicer and watch cross-filtering

1. Add a **Slicer** visual; drag `store_id` into it.
2. Add a **Card** visual; drag `quantity` onto it (shows total quantity).
3. Click a store in the slicer.

**Expected output:** both the line chart and the card instantly re-filter to that store. This is **interactivity** — the thing Excel exports fundamentally can't do, and the first business reason Power BI exists.

Also try clicking a point *on the line chart itself*: other visuals highlight to that date. Visuals filter each other by default ("cross-filtering"). Click empty whitespace to clear.

### Step 4 — Now watch the flat file break (three failures)

**Failure 1 — The business speaks names, not IDs.** The CFO asks: "Show me quantity **by region**." Look at the Data pane: `sales` has `store_id` but no region — region lives in `stores.csv`, which you didn't load. A flat fact table can't answer the most basic business grouping. *(Fixed in Lab_03 with relationships.)*

**Failure 2 — There is no revenue.** Try to answer "what was revenue?" Revenue is `quantity × unit_price × (1 − discount_pct)` **per row, then summed**. Summing each column first and multiplying the totals gives a badly wrong number (multiply-then-sum ≠ sum-then-multiply). Prove it to yourself:
   - Card 1: drag `quantity` and note the total. Card 2: `unit_price` — notice the default aggregation of a price column is a *sum of prices*, a meaningless number. There is no drag-and-drop path to correct revenue. *(Fixed in Lab_04 with measures and iterators.)*

**Failure 3 — The data is dirty.** In **Table view**, select the `sales` table, click the `customer_id` column header and sort ascending: **blank customer_ids** float to the top (~2% of rows, planted in Lab_00). Sort `quantity` ascending: **negative quantities** (returns entered as sales). Any count of customers or sum of quantities is silently wrong. *(Fixed in Lab_02 with Power Query.)*

**Verification for this step:** you can state all three failures out loud, and you have seen each with your own eyes in the file. Seriously — say them out loud. They are interview answers.

### Step 5 — Title the page and save

1. Double-click the page tab (bottom-left) → rename to `Naive Sales View`.
2. Insert a **Text box** at the top: "Lab 01 — naive flat-file report. Known issues: no region, no revenue measure, dirty rows." Dating and labeling known-bad artifacts is a professional habit — nobody who finds this file later should mistake it for the real thing.
3. **Ctrl+S**.

**Verification:** file exists at `C:\PowerBI_Labs\NorthStar\pbix\NorthStar_Lab01.pbix` and reopens without errors.

---

## 5. Production Engineering Practices

Introduced this lab; each grows through the module.

- **Naming & versioning.** One artifact per lab (`NorthStar_Lab01.pbix`, `NorthStar_Lab02.pbix`, ...), meaningful page names, labeled known issues. `.pbix` files are binary — Git can store but not meaningfully diff them — so **file-name versioning plus a change log** is the standard Desktop-era discipline. *Failure story:* a consultancy delivered `Final_v2_FINAL_new(3).pbix` to a client; three months later nobody, client or consultancy, could say which file was actually in production. The rework cost a week. Names are cheap; archaeology is not.
- **Know your artifact.** You can now explain what lives inside a `.pbix` (queries, model, report). Engineers who treat the file as a black box can't reason about size, refresh time, or what a change affects.
- **Fail loudly, not silently.** The naive report *renders* — it just answers questions wrongly. The most dangerous BI failure mode is not an error message; it's a plausible wrong number. This is why Lab_02 quarantines bad rows visibly instead of deleting them, and why the capstone requires an audit table (FR-03).
- **Reproducibility.** Your data came from a seeded script (Lab_00), your file has a deterministic name, and every step here is written down. If your laptop died tonight, you could rebuild this lab in 20 minutes. Aim to keep that property through the whole module.

---

## 6. Reflection

### What you learned

- What BI is, who consumes it, and where it sits after the SK-01..05 pipeline layers
- The five Power BI ecosystem components and the Desktop-vs-Service mental model
- What a `.pbix` contains: queries, semantic model, report — three layers, three sets of labs
- Visuals are implicit `GROUP BY` queries; default aggregations; cross-filtering
- Three concrete ways flat-file reporting fails: no dimensional context, no correct calculations, dirty data

### Interview questions (answer out loud first)

1. **What is the difference between Power BI Desktop and the Power BI Service?**
   *Desktop is the free Windows authoring tool where you build queries, the model, and the report inside a .pbix. The Service is the cloud portal for sharing, scheduled refresh, and collaboration. Build in Desktop, publish to the Service.*
2. **What are the three main layers inside a .pbix file?**
   *Power Query (data ingestion/transformation, M language), the semantic model (in-memory data, relationships, DAX measures), and the report (visuals). Changes in each layer have different blast radii — a query change alters the data itself; a report change is cosmetic.*
3. **What is a semantic model and why does it matter?**
   *A business-friendly layer of tables, relationships, and named calculations that sits between raw data and visuals. It matters because it centralizes definitions — "revenue" is defined once, correctly, and every report reuses it — instead of every spreadsheet reinventing it.*
4. **Import vs DirectQuery — when would you choose each?**
   *Import copies data into the compressed in-memory engine: fast queries, scheduled refresh, the default for most analytics. DirectQuery queries the source live: needed when data must be up-to-the-minute or is too large to import, at the cost of performance and some DAX limitations.*
5. **Why can't you compute revenue correctly by dragging quantity and unit_price into a visual?**
   *Because revenue is a row-level product (qty × price × (1 − discount)) that must be computed per row and then summed. Default aggregations sum each column independently; sum-of-products ≠ product-of-sums. You need a measure with an iterator (SUMX), covered in DAX.*
6. **A stakeholder says "the dashboard is wrong." What's your first move?**
   *Reproduce the number, then trace the layer: is the source data wrong (pipeline), the transformation wrong (Power Query), the calculation wrong (DAX), or the visual/filter context misleading (report)? The three-layer architecture gives you the debugging path.*
7. **Trap: "Power BI is just a charting tool, right?"**
   *No — charting is the last layer. The differentiator is the semantic model: compressed columnar storage, relationships, and a calculation language (DAX). Treating it as a charting tool is exactly what produces the slow, wrong reports this lab demonstrated.*

### Common interview traps

- Confusing Power BI Desktop (free, authoring) with Power BI Pro (a *license* for sharing via the Service).
- Saying "I'd fix dirty data in the visual/DAX layer" — cleansing belongs upstream in Power Query (or further upstream in the pipeline), where it runs once per refresh, not per query.

### Key takeaways

1. BI is the last mile; if it's wrong, the whole platform is "wrong" to the business.
2. A `.pbix` = queries + model + report. Keep the layers straight and debugging becomes tractable.
3. Flat files fail in predictable ways — and each failure maps to a lab: dirty data → Lab_02, no dimensions → Lab_03, no calculations → Labs 04–05, no polish/security → Lab_06.

**Next up:** [Lab_02_Power_Query_Data_Ingestion_and_Transformation.md](Lab_02_Power_Query_Data_Ingestion_and_Transformation.md) — ingest all six NorthStar files properly and clean the defects you just found.
