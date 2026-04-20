# Module D: EMR, Athena, Glue & AI-Assisted Engineering

**Time:** 6-8 hours | **Focus:** Glue ETL, Athena optimization, AI-assisted development

---

## D.1 AWS Glue

### Read

| Resource | Time | Why |
|----------|------|-----|
| [AWS Glue Developer Guide — ETL Jobs](https://docs.aws.amazon.com/glue/latest/dg/author-job.html) | 1 hr | GlueContext, DynamicFrames, job bookmarks |

### Template: Glue ETL Job

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

# Read from Catalog with bookmark
df = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders",
    transformation_ctx="source"  # Required for bookmarks
).toDF()

# Transform
clean_df = df.filter(df.amount > 0)

# Write
glueContext.write_dynamic_frame.from_options(
    frame=DynamicFrame.fromDF(clean_df, glueContext, "clean"),
    connection_type="s3",
    connection_options={"path": f"s3://bucket/clean/{args['process_date']}/"},
    format="parquet"
)

job.commit()
```

---

## D.2 Amazon Athena

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Athena Performance Tuning](https://docs.aws.amazon.com/athena/latest/ug/performance-tuning.html) | 25 min | 4 techniques to reduce cost |

### Cost Reduction Techniques

| Technique | Impact | How |
|-----------|--------|-----|
| Partitioning | 10-100x | WHERE year=2024 uses partition |
| Columnar format | 5-10x | Parquet instead of CSV |
| Compression | 2-5x | Snappy or GZIP |
| Partition projection | Faster queries | No Glue Catalog lookup |

### Query Best Practices

```sql
-- Use partition columns in WHERE
SELECT * FROM orders
WHERE year = 2024 AND month = 3;

-- Project only needed columns
SELECT order_id, amount FROM orders;  -- NOT SELECT *

-- Use LIMIT for exploration
SELECT * FROM orders LIMIT 100;
```

---

## D.3 Glue vs EMR Decision

| Factor | Glue | EMR |
|--------|------|-----|
| Setup | Zero — serverless | Cluster config needed |
| Cost | Higher per hour | Lower at scale |
| Control | Limited | Full Spark control |
| Best for | Standard ETL | Complex workloads, ML |

---

## D.4 AI-Assisted AWS Engineering

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Prompt Engineering — Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) | 30 min | Structured prompts for AWS tasks |

### AI Output Verification

```
AFTER EVERY AI-GENERATED AWS RESOURCE — VERIFY:
□ CloudFormation: run cfn-lint before deploying
□ IAM Policies: check for "Action": "*" or "Resource": "*"
□ IAM Policies: run through IAM Policy Simulator
□ Redshift DDL: verify DISTKEY is evenly distributed
□ Any ARNs or account IDs: ALWAYS replace with real values
□ Any prices: verify against AWS Pricing Calculator
```

### Trust Levels

```
USE AI FREELY:
✓ CloudFormation boilerplate
✓ IAM policy drafts (always audit)
✓ Redshift DDL with encoding
✓ boto3 script scaffolding

ALWAYS VERIFY:
⚠ IAM policies — Action/Resource "*" common errors
⚠ Redshift dist/sort key choices
⚠ Lifecycle transition minimum days

NEVER TRUST:
✗ ARNs (always contain wrong account ID)
✗ Specific AWS pricing
✗ Reserved Instance pricing
```

---

## Checkpoint

Before the Lab, you should be able to:

- [ ] Write a Glue ETL job with bookmarks
- [ ] Optimize Athena queries for cost
- [ ] Choose between Glue and EMR
- [ ] Verify AI-generated AWS configurations
