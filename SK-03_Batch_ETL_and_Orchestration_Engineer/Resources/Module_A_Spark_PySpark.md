# Module A: Apache Spark Architecture & PySpark Fundamentals

**Time:** 8-10 hours | **Focus:** Spark internals, DataFrames, core transformations

---

## A.1 Spark Architecture

### Read (Choose ONE)

| Resource | Time | Why |
|----------|------|-----|
| [Spark: The Definitive Guide — Ch 1-4](https://www.oreilly.com/library/view/spark-the-definitive/9781491912201/) | 2 hrs | Authoritative reference on Spark cluster model, execution, DataFrames |
| [Understanding Spark's Execution Model — Databricks](https://www.databricks.com/blog/2015/06/22/understanding-your-spark-application-through-visualization.html) | 20 min | Visual Job → Stage → Task hierarchy |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [PySpark Full Tutorial — freeCodeCamp](https://www.youtube.com/watch?v=_C8kWso4ne4) | 3 hrs | Comprehensive tutorial covering all core operations |

---

## A.2 Core PySpark Operations

### Reference

| Resource | Why |
|----------|-----|
| [PySpark DataFrame API Reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/dataframe.html) | Bookmark this — select, filter, withColumn, groupBy, join |

### Template: Core DataFrame Operations

```python
from pyspark.sql import functions as F

# ── READING ───────────────────────────────────────────────
df = spark.read.schema(my_schema).json("s3://bucket/path/")
df = spark.read.parquet("s3://bucket/path/")
df = spark.read.option("header", True).csv("path/")

# ── INSPECTING ────────────────────────────────────────────
df.printSchema()            # Column names and types
df.show(5, truncate=False)  # First 5 rows
df.count()                  # Row count (triggers job)

# ── FILTERING ─────────────────────────────────────────────
df.filter(F.col("qty") > 0)
df.filter(F.col("status").isin(["A", "B"]))
df.filter(F.col("name").isNotNull())

# ── TRANSFORMING ──────────────────────────────────────────
df.withColumn("new_col", F.col("a") * F.col("b"))
df.withColumn("flag", F.when(F.col("qty") < 0, True).otherwise(False))
df.withColumnRenamed("old", "new")

# ── AGGREGATING ───────────────────────────────────────────
df.groupBy("region").agg(
    F.sum("revenue").alias("total_revenue"),
    F.countDistinct("customer_id").alias("unique_customers"),
)

# ── JOINING ───────────────────────────────────────────────
orders.join(products, on="product_id", how="left")
orders.join(F.broadcast(dim_store), on="store_id", how="inner")

# ── WRITING ───────────────────────────────────────────────
df.write.mode("overwrite").partitionBy("year", "month").parquet("s3://out/")
```

---

## A.3 Data Formats & Delta Lake

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Delta Lake Quick Start](https://docs.delta.io/latest/quick-start.html) | 30 min | Standard output format for modern Spark pipelines |

### Format Selection Guide

| Use Case | Format | Why |
|----------|--------|-----|
| Analytics/Data Lake | **Parquet** or **Delta** | Columnar, compressed |
| Streaming | **Delta** | ACID transactions |
| Business users | **CSV** | Excel compatible |
| APIs | **JSON** | Web native |

---

## A.4 Spark Configuration

### Template: Production Spark Config

```python
SparkSession.builder
# Adaptive Query Execution (enable for production)
.config("spark.sql.adaptive.enabled", "true")
.config("spark.sql.adaptive.coalescePartitions.enabled", "true")
.config("spark.sql.adaptive.skewJoin.enabled", "true")

# Shuffle partitions — tune to data size
.config("spark.sql.shuffle.partitions", "200")

# Broadcast join threshold (increase for medium dims)
.config("spark.sql.autoBroadcastJoinThreshold", "50mb")

# Dynamic partition overwrite
.config("spark.sql.sources.partitionOverwriteMode", "dynamic")
```

---

## Checkpoint

Before moving to Module B, you should be able to:

- [ ] Explain the Job → Stage → Task hierarchy
- [ ] Write basic PySpark transformations and aggregations
- [ ] Read and write Parquet/Delta files
- [ ] Configure a SparkSession for production
