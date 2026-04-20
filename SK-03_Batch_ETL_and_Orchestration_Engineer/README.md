# SK-03: Batch ETL and Orchestration Engineer

> Build production Spark pipelines and orchestrate workflows with Airflow and dbt.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 40-50 hours |
| **Prerequisites** | SK-02 SQL and Data Modeling Specialist |
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
│   RESOURCES              LAB                 CAPSTONE           │
│   (30-35 hrs)            (8-10 hrs)          (12-15 hrs)        │
│                                                                 │
│   Module A: Spark Architecture & PySpark Fundamentals           │
│        ↓                                                        │
│   Module B: Airflow Orchestration                               │
│        ↓                                                        │
│   Module C: dbt Transformations                                 │
│        ↓                                                        │
│   Module D: Pipeline Patterns & Production Practices            │
│        ↓                                                        │
│   Guided Lab: Retail Sales Pipeline (Spark + Airflow)           │
│        ↓                                                        │
│   Capstone: Supply Chain ETL Platform                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Focus Areas |
|--------|-------------|
| A | Spark architecture, DataFrames, transformations, actions, Spark UI |
| B | DAG design, task dependencies, XComs, sensors, scheduling |
| C | dbt models, refs, sources, tests, documentation |
| D | Idempotency, incremental loads, partitioning, error handling |

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

If this looks unfamiliar, ensure you've completed SK-01 and SK-02 first.

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Spark, Airflow, and dbt learning materials |
| [Labs/](Labs/) | Guided retail sales pipeline |
| [Capstone/](Project/) | Supply chain ETL platform |

---

## Badge Criteria

To earn the **Batch ETL and Orchestration Engineer** badge:

1. Complete all **Essential** resources
2. Complete the guided Lab
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
