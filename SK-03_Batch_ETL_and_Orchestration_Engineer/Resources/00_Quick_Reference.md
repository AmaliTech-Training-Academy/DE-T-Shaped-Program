# SK-03 Quick Reference

PySpark, Airflow, and dbt essentials.

---

## PySpark DataFrame API

```python
from pyspark.sql import functions as F

# Reading
df = spark.read.parquet("s3://bucket/path/")
df = spark.read.option("header", True).csv("path/")

# Transformations
df.filter(F.col("amount") > 0)
df.withColumn("year", F.year(F.col("date")))
df.withColumnRenamed("old", "new")
df.drop("unwanted_col")

# Aggregations
df.groupBy("region").agg(
    F.sum("revenue").alias("total"),
    F.count("*").alias("count")
)

# Joins
orders.join(products, on="product_id", how="left")

# Writing
df.write.mode("overwrite").partitionBy("year", "month").parquet("output/")
```

---

## Airflow DAG Pattern

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG(
    dag_id="my_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule_interval="@daily",
    catchup=False,
) as dag:

    extract = PythonOperator(task_id="extract", python_callable=extract_fn)
    transform = PythonOperator(task_id="transform", python_callable=transform_fn)
    load = PythonOperator(task_id="load", python_callable=load_fn)

    extract >> transform >> load
```

---

## dbt Model Structure

```
dbt_project/
├── models/
│   ├── staging/          # 1:1 with sources
│   │   └── stg_orders.sql
│   ├── intermediate/     # Business logic
│   │   └── int_orders_enriched.sql
│   └── marts/            # Final outputs
│       └── fct_orders.sql
└── dbt_project.yml
```

### Model Example

```sql
-- models/marts/fct_orders.sql
{{ config(materialized='incremental', unique_key='order_id') }}

SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM {{ ref('int_orders_enriched') }}
{% if is_incremental() %}
WHERE order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

---

## Idempotency Rules

1. **Same input → Same output** (every run)
2. **Partition overwrite** not full table
3. **MERGE/UPSERT** not INSERT
4. **Date-based folders** for outputs

---

## Spark Performance Tips

- Never `collect()` large DataFrames
- Use `broadcast()` for small dimension tables
- Partition outputs by query filter columns
- Avoid `repartition()` unless necessary
