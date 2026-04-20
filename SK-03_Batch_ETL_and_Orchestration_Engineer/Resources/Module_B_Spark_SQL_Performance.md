# Module B: Advanced PySpark, Spark SQL & Performance

**Time:** 6-8 hours | **Focus:** Catalyst optimizer, partitioning, performance tuning

---

## B.1 Spark SQL & Catalyst Optimizer

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Catalyst Optimizer — Databricks](https://www.databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html) | 30 min | 4-stage optimization pipeline with examples |
| [Adaptive Query Execution — Databricks](https://docs.databricks.com/optimizations/aqe.html) | 20 min | AQE's three optimizations: coalesce, skew join, broadcast |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Spark SQL Deep Dive — Spark Summit](https://www.youtube.com/watch?v=GDeePbbCz2g) | 45 min | Catalyst explained by its designers |

---

## B.2 Partitioning & Shuffling

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Spark Performance Tuning — Official Docs](https://spark.apache.org/docs/latest/sql-performance-tuning.html) | 30 min | Shuffle config, broadcast thresholds, join hints |
| [Data Skew in Spark — Databricks](https://www.databricks.com/blog/2021/12/06/skew-join-optimization-in-databricks-runtime.html) | 25 min | Diagnosis, AQE handling, manual salting |

### Repartition vs Coalesce Decision Guide

```
Use REPARTITION when:
✓ Increasing partition count
✓ Need equal-size partitions
✓ Hash-distribute by key for downstream join
  df.repartition(200, F.col("store_id"))
Cost: Full shuffle

Use COALESCE when:
✓ Reducing partition count
✓ Before write to avoid small files
  df.coalesce(10).write.parquet(path)
Cost: No shuffle

RULE: coalesce(n) where n = ceil(data_size_bytes / 128MB)
```

---

## B.3 Performance Diagnosis

### Flowchart

```
Job is slow. Where do I start?
│
├── Open Spark UI → Jobs tab
│   └── Which stage is the slowest?
│
├── Click slowest stage → Tasks tab
│   ├── One task 10× slower?
│   │   YES → DATA SKEW
│   │   Fix: AQE skew join or manual salting
│   │
│   └── All tasks slow?
│       YES → SHUFFLE or PARTITION SIZE
│       Fix: broadcast small table, filter before join
│
├── Tasks spilling to disk?
│   YES → MEMORY PRESSURE
│   Fix: increase executor memory, uncache unused DFs
│
└── Stage 1 slow (no shuffle reads)?
    YES → SCAN PERFORMANCE
    Fix: column pruning, predicate pushdown
```

---

## B.4 Data Quality Patterns

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Great Expectations — Core Concepts](https://docs.greatexpectations.io/docs/conceptual_guides/gx_overview) | 30 min | Industry standard for pipeline data quality |

### Pattern: Dead Letter Queue

```python
# Separate good and bad records
valid_df = df.filter(F.col("amount") > 0)
invalid_df = df.filter(F.col("amount") <= 0)

# Write valid to target
valid_df.write.parquet("s3://bucket/clean/")

# Write invalid to dead letter
invalid_df.withColumn("rejection_reason", F.lit("invalid_amount")) \
    .write.parquet("s3://bucket/dead_letter/")
```

---

## Checkpoint

Before moving to Module C, you should be able to:

- [ ] Read and interpret Spark execution plans
- [ ] Choose between repartition and coalesce
- [ ] Diagnose data skew using Spark UI
- [ ] Implement a dead letter queue pattern
