# SK-07 RESOURCES: Microsoft Fabric Data Engineer

---

## How to Use This Resource Guide

Items marked **(Essential)** should be completed by all participants. Items marked **(Recommended)** deepen understanding. Items marked **(Advanced)** target participants pursuing project-ready mastery. All Microsoft documentation links are free.

---

---

## MODULE A: Microsoft Fabric Platform & OneLake

---

### A.1–A.2: Fabric Architecture and OneLake

#### Articles

1. **(Essential)** [Microsoft Fabric Documentation — Overview](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview)
   - **Why:** The official starting point. Explains all Fabric experiences, the OneLake concept, capacity tiers, and licensing. Start here before anything else — understanding the platform context prevents confusion between Lakehouse, Warehouse, and SQL Endpoint later.
   - **Time:** 30 min read

2. **(Essential)** [OneLake: The OneDrive for Data](https://learn.microsoft.com/en-us/fabric/onelake/onelake-overview)
   - **Why:** The clearest official explanation of OneLake's architecture — how every Fabric item stores data in one unified place, how shortcuts work, and what the OneLake file path structure looks like. Understanding this unlocks the rest of the platform.
   - **Time:** 20 min read

3. **(Essential)** [OneLake Shortcuts — Official Docs](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts)
   - **Why:** Shortcuts are the core Fabric data-sharing mechanism. This covers all shortcut types (OneLake-to-OneLake, ADLS Gen2, S3, GCS), creation steps, and the read-only constraint. Essential for the Gold → Warehouse serving layer pattern.
   - **Time:** 25 min read

4. **(Recommended)** [Fabric vs Azure Data Services — Decision Guide](https://learn.microsoft.com/en-us/fabric/get-started/fabric-vs-azure)
   - **Why:** Provides the official Microsoft view on when to use Fabric vs keeping existing Azure services. Covers Fabric Pipelines vs ADF, Fabric Spark vs Databricks, and Fabric Warehouse vs Synapse. Essential for client conversations.
   - **Time:** 20 min read

#### Videos

1. **(Essential)** [Microsoft Fabric Full Course — Enterprise DNA](https://www.youtube.com/watch?v=5rdo0r0RyNU)
   - **Why:** The most comprehensive Fabric walkthrough available (4 hours). Covers Lakehouse, Pipelines, Notebooks, Warehouse, Real-Time Intelligence, and Power BI integration with live demos. Treat this as a visual orientation to the platform before the hands-on modules.
   - **Duration:** 240 min (watch in sections alongside the labs)

2. **(Recommended)** [Fabric Lakehouse Deep Dive — Guy in a Cube](https://www.youtube.com/watch?v=JCp9m5yzGrU)
   - **Why:** Focused walkthrough of the Lakehouse — creating tables, Files vs Tables folder, SQL Analytics Endpoint, and how Power BI connects. More practical than the full course for the Lakehouse modules.
   - **Duration:** 60 min

---

### A.3: Delta Lake in Fabric

#### Articles

1. **(Essential)** [Delta Lake Tables in Fabric — Official Docs](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-and-delta-tables)
   - **Why:** The Fabric-specific Delta documentation: how Delta tables are stored in OneLake, how to read/write from notebooks and the SQL endpoint, and Delta maintenance (OPTIMIZE, VACUUM) in the Fabric context.
   - **Time:** 25 min read

2. **(Essential)** [Delta Lake MERGE Documentation — Delta.io](https://docs.delta.io/latest/delta-update.html)
   - **Why:** The authoritative reference for Delta MERGE syntax — all whenMatched/whenNotMatched conditions, conditional updates, the merge predicate. Required for every Bronze-to-Silver and Silver-to-Gold notebook that uses upsert patterns.
   - **Time:** 20 min read

3. **(Recommended)** [Delta Lake Time Travel — Delta.io](https://docs.delta.io/latest/delta-batch.html#query-an-older-snapshot-of-a-table-time-travel)
   - **Why:** Full reference for time travel: version-based and timestamp-based queries, DESCRIBE HISTORY output, RESTORE command syntax, and retention configuration. Use alongside the A.3 lab.
   - **Time:** 15 min read

#### Custom Resources

1. **(Essential)** **Delta Lake Operations Quick Reference**

   ```
   WRITE MODES:
   ─────────────────────────────────────────────────────────────────
   mode("overwrite")             → truncate and replace entire table
   mode("append")                → add rows without touching existing
   option("replaceWhere", cond)  → overwrite only rows matching condition
                                   (partition-scoped — much faster than full overwrite)
   MERGE                         → upsert (update matching rows, insert new ones)

   WHEN TO USE EACH:
   ─────────────────────────────────────────────────────────────────
   Bronze ingestion:             APPEND (raw data is immutable — never overwrite)
   Silver daily load:            MERGE (handles updates from source) or
                                 replaceWhere (partition-scoped reload if no updates)
   Gold dimension (small, static): mode("overwrite") or MERGE
   Gold fact (incremental):      MERGE on business key

   MAINTENANCE:
   ─────────────────────────────────────────────────────────────────
   OPTIMIZE table                → compact small files to 128MB–1GB targets
   OPTIMIZE table ZORDER BY (a,b)→ compact + sort data within files by a,b
   VACUUM table                  → delete files outside retention window (default 7 days)
   VACUUM table RETAIN N HOURS   → explicitly set retention before vacuuming

   SAFETY RULES:
   ─────────────────────────────────────────────────────────────────
   ✓ Run OPTIMIZE after every MERGE (MERGE creates small files)
   ✓ Wait 30 minutes after OPTIMIZE before VACUUM (active queries may reference old files)
   ✓ Never use overwriteSchema without running Impact Analysis first
   ✓ Never VACUUM with < 168 hours retention in production (can't time-travel past it)
   ✗ Never VACUUM immediately after OPTIMIZE — wait for active queries

   TIME TRAVEL:
   ─────────────────────────────────────────────────────────────────
   spark.read.format("delta").option("versionAsOf", 5).table("mydb.mytable")
   spark.read.format("delta").option("timestampAsOf", "2026-03-01").table("mydb.mytable")
   DESCRIBE HISTORY mydb.mytable        -- shows all versions
   RESTORE TABLE mydb.mytable TO VERSION AS OF 3   -- roll back
   ```

---

---

## MODULE B: Lakehouse Architecture & Data Pipelines

---

### B.1–B.2: Medallion Architecture and Pipelines

#### Articles

1. **(Essential)** [Medallion Architecture in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/onelake/onelake-medallion-lakehouse-architecture)
   - **Why:** The official Microsoft guidance on implementing Bronze/Silver/Gold in Fabric. Covers the decision between 1 Lakehouse vs 3 separate Lakehouses, table vs files for each layer, and the recommended patterns for each layer.
   - **Time:** 25 min read

2. **(Essential)** [Data Factory in Microsoft Fabric — Overview](https://learn.microsoft.com/en-us/fabric/data-factory/)
   - **Why:** The Fabric Data Factory (Pipelines) documentation hub. Start with the overview then drill into Copy Activity, pipeline parameters, and scheduling. The Fabric Pipelines docs are the reference for all Module B.2 activities.
   - **Time:** 30 min read (overview + Copy Activity sections)

3. **(Recommended)** [Fabric Pipeline Copy Activity — Connector Reference](https://learn.microsoft.com/en-us/fabric/data-factory/connector-overview)
   - **Why:** Lists all available connectors with their configuration properties. Use this when you need to configure a specific source (SFTP, REST, OData, SAP, Dynamics) and need the exact configuration parameters.
   - **Time:** Reference — search as needed

#### Videos

1. **(Essential)** [Data Pipelines in Microsoft Fabric — Full Tutorial](https://www.youtube.com/watch?v=vMHi8V-CjEM)
   - **Why:** End-to-end walkthrough of building a Fabric Pipeline with Copy Activity, Notebook Activity, parameterization, and scheduling. Shows exactly what the UI looks like and where each setting lives.
   - **Duration:** 45 min

2. **(Recommended)** [Fabric Spark Notebooks Deep Dive](https://www.youtube.com/watch?v=H2WmEL5lJEY)
   - **Why:** Focuses on Fabric Spark specifically — the starter pool vs custom pool trade-offs, attaching to Lakehouses, notebookutils API, and debugging tips in the Fabric notebook UI.
   - **Duration:** 40 min

#### Custom Resources

1. **(Essential)** **Medallion Layer Design Checklist**

   ```
   BEFORE WRITING ANY NOTEBOOK — ANSWER THESE:

   BRONZE LAYER:
   □ What is the ingestion grain? (One file per day? One row per API response?)
   □ What partition column will be used? (Usually _ingested_at cast to date)
   □ Which metadata columns are needed? (_source_system, _ingested_at, _file_name)
   □ What is the retention policy? (90 days minimum for reprocessing)
   □ Is the data append-only or can it be overwritten? (Bronze = always append)

   SILVER LAYER:
   □ What is the business key for deduplication? (natural key from source)
   □ Are there multiple sources that need conforming? (standardise codes first)
   □ What are the quality rules? (at least: not-null on PK, positive amounts, valid dates)
   □ Is MERGE needed (records updated after initial load) or replaceWhere (daily reload)?
   □ What partition column makes business sense? (date, region, status)

   GOLD LAYER:
   □ What is the grain of each fact table? (One row per order LINE, not per order)
   □ What are the 1–2 most common WHERE/JOIN columns? (These are the ZORDER columns)
   □ Are any measures semi-additive? (headcount, balance, inventory → use LASTDATE)
   □ Which Gold columns are "public API" downstream (Warehouse views reference them)?
   □ Has Impact Analysis been run before any schema change?
   ```

2. **(Essential)** **Pipeline Expression Quick Reference**

   ```
   COMMON PIPELINE EXPRESSIONS (Dynamic Content):
   ─────────────────────────────────────────────────────────────────
   Yesterday's date (default processDate):
     @formatDateTime(addDays(utcNow(), -1), 'yyyy-MM-dd')

   Dynamic file path with date:
     @concat('exports/orders_', pipeline().parameters.processDate, '.csv')

   Rows copied from a Copy Activity:
     @activity('Copy_Orders').output.rowsCopied

   Pipeline run ID:
     @pipeline().RunId

   Current pipeline name:
     @pipeline().Pipeline

   Pass parameter to Notebook:
     Set "process_date" → @pipeline().parameters.processDate

   Build Teams alert body:
     @concat('Pipeline FAILED\nName: ', pipeline().Pipeline,
             '\nDate: ', pipeline().parameters.processDate,
             '\nError: ', activity('My_Activity').error.message)

   EVALUATION TIPS:
   □ Test expressions in the Expression Builder before deploying
   □ Use @{} syntax for string interpolation inside longer strings
   □ @utcNow() is UTC — adjust for client timezone with convertTimeZone()
   □ Date arithmetic: addDays(utcNow(), -7) for 7 days ago
   ```

---

### B.3–B.4: PySpark Notebooks

#### Articles

1. **(Essential)** [PySpark Documentation — DataFrame API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html)
   - **Why:** The reference for all PySpark DataFrame operations used in Bronze-to-Silver notebooks. Bookmark the functions module — withColumn, filter, groupBy, join, dropDuplicates, and window functions are the most-used operations.
   - **Time:** Reference — use as needed

2. **(Recommended)** [notebookutils API in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/notebook-utilities)
   - **Why:** The Fabric-specific utilities API — widgets (for receiving pipeline parameters), notebook exit (for returning output to the pipeline), file system operations (for manipulating OneLake files), and secrets (for accessing Key Vault). The `dbutils.widgets.get()` pattern for parameters is critical.
   - **Time:** 20 min read

---

---

## MODULE C: Fabric Warehouse, Dataflows Gen2 & Analytics

---

### C.1–C.2: Fabric Warehouse

#### Articles

1. **(Essential)** [Fabric Data Warehouse Overview — Official Docs](https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehousing)
   - **Why:** The Fabric Warehouse architecture documentation — how T-SQL runs on Delta files in OneLake, the difference from Azure Synapse, DDL support (what T-SQL features work vs what doesn't), and when to use the Warehouse vs the Lakehouse SQL Endpoint.
   - **Time:** 25 min read

2. **(Recommended)** [Cross-Database Queries in Fabric](https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse)
   - **Why:** Shows the three-part name syntax for cross-database queries in Fabric, how to join Lakehouse tables and Warehouse tables in the same query, and performance characteristics. Required for the shortcut-based serving layer architecture.
   - **Time:** 15 min read

---

### C.3–C.4: Dataflows Gen2 and Power BI

#### Articles

1. **(Essential)** [Dataflows Gen2 in Microsoft Fabric — Official Docs](https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-overview)
   - **Why:** Complete Dataflow Gen2 documentation — architecture, supported destinations, incremental refresh configuration, and the difference from Power BI Dataflows Gen1. The destination configuration section is essential for Module C.3.
   - **Time:** 25 min read

2. **(Essential)** [DirectLake in Microsoft Fabric — Official Docs](https://learn.microsoft.com/en-us/fabric/get-started/direct-lake-overview)
   - **Why:** The definitive explanation of DirectLake mode — how it works architecturally, the row limits per SKU, when fallback to DirectQuery occurs, and how to set up a DirectLake semantic model. Required for the Power BI integration module.
   - **Time:** 20 min read

3. **(Recommended)** [Incremental Refresh for Dataflows Gen2](https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-incremental-refresh)
   - **Why:** Step-by-step instructions for configuring RangeStart/RangeEnd parameters, setting the refresh policy, and verifying that incremental refresh is working. The exact parameter names (case-sensitive) and folding requirement are covered.
   - **Time:** 20 min read

---

---

## MODULE D: Real-Time Intelligence, Governance & AI-Assisted Engineering

---

### D.1–D.2: Real-Time Intelligence

#### Articles

1. **(Essential)** [Eventstreams in Microsoft Fabric — Official Docs](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/event-streams/overview)
   - **Why:** Complete Eventstreams documentation — source types, destination types, no-code transformations, and Event Hub configuration. The consumer group requirement and offset position settings are easy to miss without reading this first.
   - **Time:** 25 min read

2. **(Essential)** [KQL Quick Reference — Microsoft Docs](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
   - **Why:** One-page reference for all KQL operators and functions. Bookmark this — use it constantly when writing KQL queries. The summarize, arg_max, bin(), make-series, and percentile entries are the most-used in data engineering.
   - **Time:** Reference — read the table once, then bookmark

3. **(Recommended)** [KQL Tutorial — Microsoft Docs](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/tutorial)
   - **Why:** Interactive KQL tutorial with progressively complex exercises using a sample dataset. The best way to build KQL fluency is to do these exercises — the time series and aggregation sections correspond directly to Module D.2 labs.
   - **Time:** 2 hours (complete the exercises)

4. **(Advanced)** [KQL Materialized Views — Microsoft Docs](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/management/materialized-views/materialized-view-overview)
   - **Why:** Materialized views in KQL databases are powerful for pre-aggregating streaming data. The create syntax, staleness window, and the trade-off between materialized views and Update Policies are covered here.
   - **Time:** 20 min read

#### Videos

1. **(Essential)** [Real-Time Intelligence in Microsoft Fabric](https://www.youtube.com/watch?v=oxLbmjGz_2I)
   - **Why:** End-to-end demo of Eventstreams → KQL Database → Real-Time Dashboard. Shows the Eventstream designer UI, the KQL query editor, and building a real-time dashboard tile. Essential before the D.1/D.2 labs.
   - **Duration:** 45 min

2. **(Essential)** [KQL for Beginners — Pluralsight / YouTube](https://www.youtube.com/watch?v=Pl8n6GaWEo0)
   - **Why:** Best introductory KQL video — covers the pipe syntax, key operators, time filters, and aggregations in a single 90-minute session. More practical than the documentation tutorial for absolute beginners.
   - **Duration:** 90 min

#### Custom Resources

1. **(Essential)** **KQL Pattern Library**

   ```
   CURRENT STATE (most recent event per entity):
   ─────────────────────────────────────────────────────────────────
   TableName
   | where timestamp > ago(24h)   // time window — extend if data is sparse
   | summarize arg_max(timestamp, *) by entity_id
   // arg_max(timeCol, *) returns all columns from the row with max timeCol

   TIME-SERIES AGGREGATION:
   ─────────────────────────────────────────────────────────────────
   TableName
   | where timestamp > ago(7d)
   | summarize metric = count() by bin(timestamp, 1h), category
   | render timechart

   ALERT QUERY (non-empty = alert condition met):
   ─────────────────────────────────────────────────────────────────
   TableName
   | where timestamp > ago(10m)   // ALWAYS time-bound to prevent re-alerting
   | summarize recent_count = count()
   | where recent_count < expected_threshold
   // Empty result = no alert; non-empty = alert

   MOVING AVERAGE:
   ─────────────────────────────────────────────────────────────────
   TableName
   | summarize count_per_hour = count() by bin(timestamp, 1h)
   | order by timestamp asc
   | extend moving_avg = series_rolling_avg(
       dynamic(count_per_hour), 3, 3  // 3-period centred rolling average
   )

   PERCENTILE (for latency/duration analysis):
   ─────────────────────────────────────────────────────────────────
   TableName
   | summarize
       avg_duration = avg(duration_ms),
       p50_duration = percentile(duration_ms, 50),
       p95_duration = percentile(duration_ms, 95),
       p99_duration = percentile(duration_ms, 99)
     by service_name

   COMMON MISTAKES TO AVOID:
   ─────────────────────────────────────────────────────────────────
   ✗ arg_max without time window → scans ALL historical data
     Fix: add | where timestamp > ago(N) before summarize

   ✗ Alert query without time boundary → re-alerts on stale data
     Fix: always include | where timestamp > ago(10m) in alert queries

   ✗ bin() without order → chart x-axis is unordered
     Fix: add | order by timestamp asc after bin()

   ✗ count() when you need distinct count
     Fix: dcount(column) for approximate distinct count (HyperLogLog),
          dcountif(column, condition) for conditional distinct count
   ```

---

### D.3: Governance

#### Articles

1. **(Essential)** [Workspace Roles in Microsoft Fabric — Official Docs](https://learn.microsoft.com/en-us/fabric/get-started/roles-workspaces)
   - **Why:** Definitive guide to the four workspace roles and their exact permissions. The comparison table between Admin/Member/Contributor/Viewer is essential for configuring the access control model on any project.
   - **Time:** 20 min read

2. **(Recommended)** [OneLake Data Access Roles — Official Docs](https://learn.microsoft.com/en-us/fabric/onelake/security/data-access-control-model)
   - **Why:** The new (2024) OneLake item-level security model — apply read/write permissions at the folder or table level within a Lakehouse without granting full workspace access. Essential for the "analysts can see Gold but not Bronze/Silver" requirement.
   - **Time:** 20 min read

3. **(Recommended)** [Microsoft Purview Integration with Fabric](https://learn.microsoft.com/en-us/fabric/governance/microsoft-purview-fabric-overview)
   - **Why:** How sensitivity labels, data lineage, and data classification work between Microsoft Purview and Fabric. The lineage and Impact Analysis features used in Module D.3 are covered here.
   - **Time:** 20 min read

---

### D.4: AI-Assisted Fabric Engineering

#### Articles

1. **(Essential)** [Prompt Engineering Overview — Anthropic Docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
   - **Why:** Producing correct Fabric Pipeline JSON, PySpark notebooks, and KQL queries from LLMs requires careful prompt structuring — context (table schema, partition column, MERGE key), constraints (must use replaceWhere not overwrite), and output format (complete notebook, not pseudocode). This guide covers the techniques.
   - **Time:** 30 min read

#### Custom Resources

1. **(Essential)** **Prompt Templates for Fabric Engineering Tasks**

   ```
   TEMPLATE 1: Generate a Bronze-to-Silver PySpark Notebook
   ─────────────────────────────────────────────────────────────────
   Generate a Microsoft Fabric PySpark notebook for Bronze-to-Silver
   transformation with the following specification:

   Source table: [lakehouse_name].[table_name]
   Source schema:
     [column_name]: [type]  // one per line
   Filter: read only rows where _ingested_at cast to date = process_date parameter

   Target table: [silver_lakehouse_name].[table_name]
   Target schema: [same or different]
   Update method: MERGE on [business_key_column]

   Required transformations:
     1. Deduplicate on [key]
     2. Standardise [column] values: [mapping]
     3. Validate: [column] must be > 0 — quarantine failures
     4. Derive column [name] = [formula]

   Notebook must:
   - Receive process_date as a dbutils widget parameter
   - Log row counts (raw, clean, rejected) to [log_table]
   - Exit with JSON summary using dbutils.notebook.exit()

   ─────────────────────────────────────────────────────────────────
   TEMPLATE 2: Generate a KQL Query
   ─────────────────────────────────────────────────────────────────
   Generate a KQL query for Microsoft Fabric Real-Time Intelligence.

   Table name: [table_name]
   Table schema: [columns and types]
   Business question: [what I need to know]
   Time window: last [N hours/days]
   Grouping: by [columns]
   Output format: [table / timechart / barchart]

   ─────────────────────────────────────────────────────────────────
   TEMPLATE 3: Generate Fabric Pipeline JSON
   ─────────────────────────────────────────────────────────────────
   Generate a Microsoft Fabric Data Pipeline Copy Activity configuration
   in JSON format for:

   Source type: [REST API / SFTP / OData]
   Source details: [URL, method, pagination if applicable]
   Sink: Lakehouse Table "[lakehouse_name]"."[table_name]"
   Update method: Append (or Overwrite)
   Metadata columns to add:
     _source_system = "[system_name]"
     _ingested_at = pipeline run time
   Retry: 3 retries, 60 second interval

   ─────────────────────────────────────────────────────────────────
   AFTER EVERY AI-GENERATED FABRIC OUTPUT — VERIFY:
   □ MERGE key: is it the business key (not surrogate, not row number)?
   □ replaceWhere condition: does it match the actual partition column?
   □ OPTIMIZE: is it called AFTER the MERGE (not before — no files created yet)?
   □ Metadata columns: @utcNow() (Pipeline expression) not datetime.now() (Python)
   □ KQL arg_max: does the time window cover sparse-reporting devices?
   □ KQL alert: is there a time boundary (ago(N)) to prevent stale re-alerts?
   □ Warehouse view: does it expose business-friendly column names (not technical keys)?
   □ Any hardcoded connection strings or API keys? (Must use Key Vault references)
   ```

2. **(Essential)** **Common Fabric Anti-Pattern Reference**

   ```
   ANTI-PATTERN                    PROBLEM                         FIX
   ──────────────────────────────────────────────────────────────────────────
   Bronze overwrite                Destroys historical data for     Always APPEND to Bronze
                                   reprocessing                     tables. Never overwrite.

   MERGE without partition filter  Full table scan on every MERGE   Add partition column to
                                   → slow as table grows             MERGE condition

   OPTIMIZE before MERGE           Compacts files that don't exist  Run OPTIMIZE AFTER MERGE
                                   yet — wasted compute

   VACUUM immediately after        Active queries may reference old  Wait 30 min after OPTIMIZE
   OPTIMIZE                        files → query failure            before VACUUM

   ZORDER on 3+ columns            Each additional column reduces   ZORDER on max 2 columns
                                   benefit of all others            (the 1–2 most filtered)

   SELECT * in Warehouse views     Forces reading all Parquet        Always list required columns
                                   columns — columnar storage only  explicitly
                                   skips unread columns

   arg_max without time filter     Scans entire KQL table history    Always add | where timestamp
                                   → slow + expensive                > ago(Nh) before arg_max

   Alert without time boundary     Re-alerts on old events every    Add | where timestamp >
                                   time the alert runs               ago(10m) in every alert

   Hardcoded API key in pipeline   Security vulnerability;          Use Fabric Key Vault
   JSON                            key visible to all workspace     reference or environment
                                   members                           variable
   ```
