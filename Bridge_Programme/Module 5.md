# MODULE 5: The Modern Data Stack — How Everything Connects

## Why This Module Exists

This is the map. Everything you learn in the 12 skillsets is a piece of a larger picture. Before you go deep on any individual tool, you need to understand what role it plays and how all the pieces connect. Without this map, you risk learning skills in isolation and struggling to apply them in combination.

---

## Module 5 Glossary

**Data pipeline** — A sequence of steps that move and transform data from a source to a destination. Like a factory assembly line: raw materials go in one end, finished product comes out the other.

**ETL (Extract, Transform, Load)** — The classic data pipeline pattern: extract data from a source, transform it (clean, reshape, aggregate), load it into a destination (data warehouse).

**ELT (Extract, Load, Transform)** — A modern variation: extract data from a source, load it raw into the destination, transform it inside the destination. This is what tools like dbt enable.

**Data warehouse** — A database designed for analytical queries (not transactional operations). Optimised for reading large amounts of data quickly. Examples: Snowflake, BigQuery, Amazon Redshift, Microsoft Fabric Warehouse.

**Data lake** — A storage repository that holds raw data in its original format until needed. Cheap storage, flexible formats. Usually built on object storage (S3, Azure Data Lake).

**Data lakehouse** — An architecture that combines the cheap storage of a data lake with the query capabilities of a data warehouse. Delta Lake, Apache Iceberg, and Microsoft Fabric are lakehouses.

**dbt (data build tool)** — A tool for writing SQL-based transformations that run inside a data warehouse or lakehouse. Adds version control, testing, and documentation to SQL transformations.

**Airflow** — An orchestration tool. It schedules and monitors pipelines — ensuring that step B runs after step A completes, alerting when something fails, and allowing pipelines to be retried.

**Spark** — A distributed computing engine. It can process data that is too large to fit on one computer by splitting the work across many computers. The standard engine for large-scale data transformations.

**Kafka** — A distributed streaming platform. It acts as a high-speed, durable message bus. Data producers publish events to Kafka; data consumers read and process those events in real time.

**Orchestration** — Coordinating multiple pipeline steps to run in the correct order at the correct time. Airflow and Fabric Pipelines are orchestration tools.

**Ingestion** — The act of moving data from a source system (a database, an API, a file) into the data platform. The first step of any pipeline.

**Schema** — The structure of a table: its column names, data types, and constraints.

**Medallion architecture** — A data organisation pattern with three layers: Bronze (raw ingested data), Silver (cleaned, validated data), Gold (analytics-ready aggregated data).

**Batch processing** — Processing data in large chunks at scheduled intervals (e.g., every hour, every night). The data is collected first, then processed together.

**Streaming processing** — Processing data as it arrives, event by event, with very low latency (milliseconds). Contrast with batch.

**Latency** — How long something takes. Low-latency systems respond in milliseconds. High-latency batch systems might process data every 24 hours.

**SLA (Service Level Agreement)** — A commitment about performance. "Data will be refreshed by 7am every day" is an SLA. Data engineers are responsible for meeting SLAs.

---

## Concept Explainer 5.1: The Data Engineering Job — What You Actually Do

A data engineer's job is to **build and maintain the systems that move data from where it is created to where it is used**.

Data is created by: applications (every order placed on a website, every user login, every sensor reading), external APIs (weather data, financial prices, social media), and files (reports, exports, partner data feeds).

Data is used by: analysts who run reports, data scientists who build models, executives who look at dashboards, and applications that need processed data.

The data engineer builds the pipes and plumbing that connects these two worlds.

**The three things data engineers build:**

1. **Ingestion pipelines** — Code that pulls data from source systems and drops it into the data platform (the lake or warehouse). SK-01, SK-02, SK-03, SK-04, SK-05 are primarily about this.

2. **Transformation pipelines** — Code that takes raw data and produces cleaned, validated, and enriched versions. SK-02, SK-03, SK-07, SK-08 are primarily about this.

3. **Serving layers** — The final, analytics-ready data that analysts and dashboards consume. SK-02, SK-06, SK-07 are primarily about this.

---

## Concept Explainer 5.2: Batch vs Streaming — Two Different Clocks

All data pipelines run on one of two rhythms:

**Batch:** "Process all the data from the last 24 hours, every morning at 6am."
- Data accumulates in a source
- At a scheduled time, the pipeline runs and processes the accumulated data
- Analysts see data that is up to 24 hours old
- Most pipelines are batch pipelines

**Streaming:** "Process each event as it happens, within one second."
- Data flows continuously (like a river)
- The pipeline processes events as they arrive
- Analysts see data that is seconds old
- More complex and expensive — only use it when low latency genuinely matters

**When does latency actually matter?**
- A daily sales report? Batch is fine — nobody needs sales data from the last 5 minutes.
- A fraud detection system? Streaming — you need to detect a fraudulent transaction before the money leaves.
- A recommendation engine? Usually batch — calculating recommendations every hour is sufficient.

The key question to ask on any project: "How old can this data be before it stops being useful?" The answer determines whether you need batch or streaming.

---

## Concept Explainer 5.3: The Medallion Architecture — Bronze, Silver, Gold

This is the most important structural pattern in data engineering. Every data platform you will work on will have some version of this.

**Bronze — Raw Data**
Data arrives from a source system and is stored exactly as received, with no modification. The only additions are metadata columns like "when was this ingested" and "which system sent it."

Purpose: Preserve the original data exactly as it arrived. If something goes wrong downstream, you can always reprocess from Bronze.

Example: A CSV file exported from a PostgreSQL database, stored in S3 exactly as exported, with an added column `_ingested_at` = the timestamp it arrived.

**Silver — Clean, Validated Data**
Raw data is cleaned (remove nulls, fix types, standardise values), validated (amounts must be positive, dates must be in the past), and deduplicated (remove duplicate rows). The result is a reliable, queryable version of the source data.

Purpose: A trustworthy version of the data that all downstream consumers can rely on.

Example: The raw CSV from Bronze, now with: null rows removed, dates parsed to proper timestamps, status values standardised to lowercase, and duplicates removed by keeping the most recent version of each record.

**Gold — Analytics-Ready Data**
Silver data is aggregated, joined across multiple sources, and shaped into the structure that analysts and dashboards need. Star schemas (fact tables + dimension tables) live here.

Purpose: The final product that business users consume. Optimised for query performance, organised by business domain.

Example: A `fact_daily_revenue` table that joins orders (Silver), customers (Silver), and products (Silver) into a single flat table with one row per day per region, pre-calculated with total revenue, order count, and average order value.

```
Source System (PostgreSQL)
       │
       ▼
[BRONZE] raw_orders.parquet
       │  (no changes — just ingested + metadata)
       ▼
[SILVER] clean_orders.parquet
       │  (validated, deduplicated, standardised)
       ▼
[GOLD] fact_daily_revenue.parquet
         (aggregated, joined, analytics-ready)
         │
         ▼
   Analyst / Dashboard
```

---

## Concept Explainer 5.4: The Modern Data Stack — Which Tool Does What

Here is how the tools you will learn map onto the pipeline stages:

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE DATA PLATFORM                           │
│                                                                 │
│  SOURCE          INGEST          STORE          SERVE           │
│  SYSTEMS         ─────────       ──────         ─────           │
│                                                                 │
│  PostgreSQL  ──► Airflow     ──► S3/ADLS    ──► Dashboards      │
│  Salesforce  ──► Spark       ──► Delta Lake ──► BI Tools        │
│  APIs        ──► Kafka       ──► DynamoDB   ──► ML Models       │
│  Files       ──► dbt         ──► Snowflake  ──► APIs            │
│                                                                 │
│  TOOL                WHAT IT DOES              WHICH SK          │
│  ──────              ────────────              ────────          │
│  Python/pandas       Transform data in code    SK-01             │
│  SQL/dbt             Transform data in SQL     SK-02, SK-03      │
│  Airflow             Schedule + orchestrate    SK-03             │
│  Spark/PySpark       Large-scale processing    SK-03, SK-08      │
│  AWS (S3,IAM,etc.)   Cloud infrastructure      SK-04             │
│  Kafka/Kinesis       Streaming                 SK-05             │
│  Power BI            Visualisation/BI          SK-06             │
│  Microsoft Fabric    Unified MS data platform  SK-07             │
│  Delta/Iceberg       Open table formats        SK-08             │
│  Governance tools    Data security & quality   SK-09             │
│  Docker/GitHub       CI/CD for pipelines       SK-10             │
│  DynamoDB            NoSQL database            SK-11             │
│  Spark ML            ML for data engineers     SK-12             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Module 5 Self-Check

1. A company runs a pipeline every night at midnight that processes all orders from the previous day. Is this batch or streaming? What would need to change to make it a streaming pipeline?

2. An analyst queries your data and finds that the total revenue is 15% higher than the number in last week's report, even though the underlying orders have not changed. You have Bronze, Silver, and Gold layers. At which layer do you start investigating first, and why?

3. Explain the difference between a data lake and a data warehouse. Give a scenario where a company would use both (not choose between them).

4. A new feature requires joining customer data from Salesforce with order data from PostgreSQL. Where in the medallion architecture does this join happen — Bronze, Silver, or Gold — and why?

5. Your team uses Airflow to run a pipeline that: ingests data from an API, transforms it with PySpark, loads it to S3, and runs dbt models on the result. Name the role each tool plays (ingestion, transformation, orchestration, storage, or serving).

---

## Module 5 Resources

### Read First (Free, Currently Live)
1. **The Data Engineering Cookbook — Andreas Kretz** (free PDF at github.com/andkret/Cookbook) — A free, practical guide to data engineering written by a practitioner. The "Basic Data Engineering Skills" and "Data Pipelines" chapters map directly to this module. Not perfectly polished but genuinely useful.

2. **What is the Modern Data Stack? — dbt Labs Blog** (getdbt.com/blog/future-of-the-modern-data-stack) — A clear explanation of how dbt, Airflow, data warehouses, and reverse ETL tools relate to each other. Written by one of the companies that shaped the modern data stack.

3. **Medallion Architecture — Databricks** (databricks.com/glossary/medallion-architecture) — The official explanation of the Bronze/Silver/Gold pattern from Databricks, the company that popularised it. Includes diagrams.

### Watch (Free, Currently Live)
1. **Data Engineering Introduction — Andreas Kretz** — Search "Andreas Kretz data engineering" on YouTube. Andreas runs "The Data Engineering Podcast" and produces the clearest practical explanations of the data engineering landscape. Start with "How To Become A Data Engineer."

2. **Airflow Tutorial for Beginners — CodeWithYu** — Search "CodeWithYu Apache Airflow tutorial" on YouTube. A practical introduction to Airflow that shows you what orchestration looks like in practice. Watch this before SK-03.

