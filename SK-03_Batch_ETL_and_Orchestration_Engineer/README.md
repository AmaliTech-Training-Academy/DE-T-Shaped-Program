# SK-03: Batch ETL and Orchestration Engineer

> Build production Spark pipelines and orchestrate workflows with Airflow and dbt.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 40-50 hours |
| **Prerequisites** | SK-01 Python Data Engineering Practitioner and SK-02 SQL and Data Modeling Specialist |
| **Badge Earned** | Batch ETL and Orchestration Engineer |
| **Difficulty** | Intermediate |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Write PySpark jobs that process millions of rows efficiently
- [ ] Understand Spark's execution model (Jobs, Stages, Tasks)
- [ ] Build and debug Airflow DAGs for pipeline orchestration
- [ ] Use dbt for SQL-based transformations with testing
- [ ] Design idempotent, incremental ETL pipelines
- [ ] Monitor and troubleshoot distributed data processing

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCES          GRANULAR LABS     GUIDED LAB    CAPSTONE   │
│   (reading)          (25-35 hrs)       (8-10 hrs)    (12-15 hrs)│
│                                                                 │
│   Module A: Spark Architecture & PySpark Fundamentals           │
│        ↓                                                        │
│   Module B: Advanced PySpark, Spark SQL & Performance           │
│        ↓                                                        │
│   Module C: Airflow Orchestration                               │
│        ↓                                                        │
│   Module D: dbt, Cloud ETL Platforms & AI                       │
│        ↓                                                        │
│   Granular Labs 00-08: step-by-step build of the UrbanGear      │
│   pipeline on a local Docker stack                              │
│        ↓                                                        │
│   Guided Lab: UrbanGear Daily Sales Pipeline                    │
│   (PySpark + Airflow + dbt)                                     │
│        ↓                                                        │
│   Capstone: Multi-Source Financial Data Pipeline for FinCore    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Resource file | Focus Areas |
|--------|---------------|-------------|
| A | `Module_A_Spark_PySpark.md` | Spark architecture, DataFrames, transformations, actions, Spark UI |
| B | `Module_B_Spark_SQL_Performance.md` | Spark SQL, Catalyst optimizer, partitioning, performance tuning |
| C | `Module_C_Airflow_Orchestration.md` | DAG design, task dependencies, XComs, sensors, scheduling |
| D | `Module_D_dbt_ETL_AI.md` | dbt models, refs, sources, tests, documentation; cloud ETL platforms; AI in ETL |

---

## Granular Labs

The granular labs take a complete beginner, one small step at a time, to the point where they can do the end-to-end guided lab on their own. They use the guided lab's scenario (UrbanGear), data (17-field JSON orders), names (`urbangear-data-lake`, `etl_job.py`, `urbangear_daily_sales_pipeline`, dbt project `urbangear`) and order. Everything runs locally and for free on one Docker Compose stack: MinIO stands in for S3, Postgres for Redshift, and PySpark inside Airflow for AWS Glue.

| Lab | Title | Builds | Guided lab coverage |
|-----|-------|--------|---------------------|
| 00 | [Environment Setup](Granular%20Labs/Lab_00_Environment_Setup.md) | Docker stack: Airflow 2.9.3, PySpark 3.5.1, dbt 1.8, MinIO, Postgres 16 | Prerequisites |
| 01 | [MinIO Data Lake and Sample Data](Granular%20Labs/Lab_01_MinIO_Data_Lake_and_Sample_Data.md) | Bucket, zones, JSON order generator, boto3 | Scenario, input paths |
| 02 | [PySpark Sessions, Schemas and JSON](Granular%20Labs/Lab_02_PySpark_Sessions_Schemas_and_JSON.md) | `PipelineConfig`, schema, SparkSession tuning, PERMISSIVE reads | Part 1, Steps 1.1–1.2 |
| 03 | [Data Quality Flags and Routing](Granular%20Labs/Lab_03_Data_Quality_Flags_and_Routing.md) | Volume check, quality flags, clean/rejected routing | Part 1, Steps 1.3–1.4 |
| 04 | [Transformations, Window Functions and Partitioned Writes](Granular%20Labs/Lab_04_Transformations_Windows_and_Partitioned_Writes.md) | Revenue maths, windows, metrics sidecar, dynamic overwrite, `spark-submit` | Part 1, Steps 1.4–1.6 |
| 05 | [Airflow Fundamentals: Your First DAG](Granular%20Labs/Lab_05_Airflow_Fundamentals_First_DAG.md) | Operators, `{{ ds }}`, retries, XCom, callbacks | Part 2, Step 2.1 |
| 06 | [Sensors, Hooks and the Spark Task](Granular%20Labs/Lab_06_Sensors_Hooks_and_the_Spark_Task.md) | Connections, `S3KeySensor`, `S3Hook`, Spark task, backfill | Part 2, Steps 2.2–2.3 (tasks 1–3) |
| 07 | [Quality Gate, Branching and Warehouse Load](Granular%20Labs/Lab_07_Quality_Gate_Branching_and_Warehouse_Load.md) | `BranchPythonOperator`, `TaskGroup` load, trigger rules | Part 2, Steps 2.2–2.3 (tasks 4–7) |
| 08 | [dbt Models, Tests and the Full Pipeline](Granular%20Labs/Lab_08_dbt_Models_Tests_and_Full_Pipeline.md) | dbt project, incremental models, tests, `--vars`, end-to-end run | Parts 3–4 |

---

## Prerequisites Check

Can you explain what this code does?

```python
df = spark.read.parquet("s3://bucket/sales/")
result = (df
    .filter(F.col("amount") > 0)
    .groupBy("region", "product_id")
    .agg(F.sum("amount").alias("total"))
    .orderBy(F.desc("total")))
result.write.mode("overwrite").parquet("s3://bucket/aggregated/")
```

If this looks unfamiliar, start with Granular Labs 00–02 (they assume only SK-01 Python and SK-02 SQL).

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Spark, Airflow, and dbt learning materials (Modules A–D) |
| [Granular Labs/](Granular%20Labs/) | Step-by-step labs 00–08 that prepare you for the guided lab |
| [Labs/](Labs/) | End-to-end guided lab: UrbanGear daily sales pipeline |
| [Project/](Project/) | Capstone: multi-source financial data pipeline for FinCore |

---

## Completion Criteria

To earn the **Batch ETL and Orchestration Engineer** badge:

1. Complete all **Essential** resources
2. Complete the Granular Labs and the guided Lab
3. Submit a passing Capstone project:
   - PySpark ETL jobs with proper partitioning
   - Airflow DAG with task dependencies
   - dbt models with tests
   - Incremental loading implementation
   - Error handling and monitoring

---

## Platform Decision Point

After SK-03, you'll choose a platform track:

| Path | Next Skillset | Focus |
|------|---------------|-------|
| AWS | SK-04: AWS Cloud Data Engineer | S3, Glue, Athena, Redshift |
| Microsoft | SK-06: Power BI Analytics | Power BI, then Fabric |
