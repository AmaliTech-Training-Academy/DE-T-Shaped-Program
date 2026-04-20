# Module D: dbt, Cloud ETL Platforms & AI

**Time:** 8-10 hours | **Focus:** dbt fundamentals, AWS Glue, AI-assisted development

---

## D.1 dbt Fundamentals

### Read

| Resource | Time | Why |
|----------|------|-----|
| [dbt Best Practices Guide](https://docs.getdbt.com/best-practices) | 45 min | Project structure, naming, layers, testing |
| [How We Structure Our dbt Projects](https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview) | 30 min | Staging → intermediate → marts architecture |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [dbt Fundamentals — Official Course](https://courses.getdbt.com/courses/fundamentals) | 4 hrs | Models, tests, documentation, workflow |

---

## D.2 dbt Materializations

### Decision Guide

```
TABLE
✓ Use when: BI tools query frequently, final serving layer
✓ Cost: Full rebuild on every run
✗ Avoid for: Staging models

VIEW
✓ Use when: Thin wrapper, staging, simple renames
✓ Cost: No storage, computed on query
✗ Avoid for: Expensive aggregations

INCREMENTAL
✓ Use when: Large table, new data arrives daily
✓ Cost: First run = full, subsequent = delta
✓ Requires: unique_key for merge strategy
✗ Avoid when: Changes aren't additive

EPHEMERAL
✓ Use when: Used once as CTE
✓ Cost: Zero storage, inline compiled
✗ Avoid when: Referenced by 2+ models
```

---

## D.3 dbt Commands

### Reference

```bash
# Run a specific model
dbt run --select stg_orders

# Run model + all upstream dependencies
dbt run --select +fct_daily_sales

# Run model + all downstream dependents
dbt run --select stg_orders+

# Run all models with a tag
dbt run --select tag:daily

# Build = run + test in one command
dbt build --select stg_orders+

# Preview without executing
dbt ls --select stg_orders+
```

---

## D.4 Cloud ETL (AWS Glue)

### Read

| Resource | Time | Why |
|----------|------|-----|
| [AWS Glue Developer Guide — Getting Started](https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html) | 1 hr | Data Catalog, Crawlers, ETL Jobs, Bookmarks |

### Template: Glue Job Script

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ["JOB_NAME", "process_date"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Read from Data Catalog
df = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders"
).toDF()

# Transform
clean_df = df.filter(df.amount > 0)

# Write with bookmark support
glueContext.write_dynamic_frame.from_options(
    frame=DynamicFrame.fromDF(clean_df, glueContext, "clean"),
    connection_type="s3",
    connection_options={"path": f"s3://bucket/clean/{args['process_date']}/"},
    format="parquet"
)

job.commit()
```

---

## D.5 AI-Assisted Pipeline Engineering

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Prompt Engineering — Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) | 30 min | Structured prompts for code generation |

### Verification Checklist

```
AFTER EVERY AI-GENERATED PIPELINE CODE — VERIFY:
□ All import statements reference real packages
□ All column names match the actual schema
□ JOIN conditions tested with count()
□ Jinja templates are in operator's template_fields
□ Retries and timeouts configured
□ No credentials hardcoded
□ Tested on sample of real data
```

---

## Checkpoint

Before the Lab, you should be able to:

- [ ] Structure a dbt project with staging/marts layers
- [ ] Choose the right materialization for each model
- [ ] Write an incremental model with unique_key
- [ ] Create a Glue job with job bookmarks
