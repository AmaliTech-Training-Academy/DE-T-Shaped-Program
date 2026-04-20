# SK-04 Quick Reference

AWS data services essentials.

---

## AWS Data Services Overview

| Service | Purpose | Pricing |
|---------|---------|---------|
| **S3** | Object storage (data lake) | $0.023/GB/month |
| **Glue** | Serverless ETL + Catalog | $0.44/DPU-hour |
| **Athena** | Serverless SQL on S3 | $5/TB scanned |
| **Redshift** | Data warehouse | From $0.25/node-hour |
| **Kinesis** | Real-time streaming | $0.015/shard-hour |
| **Lambda** | Event-driven compute | $0.0000166667/GB-second |

---

## S3 Data Lake Structure

```
s3://my-data-lake/
├── raw/                    # Immutable source data
│   └── orders/
│       └── year=2024/month=01/
├── processed/              # Cleaned, standardized
│   └── orders/
│       └── year=2024/month=01/
└── curated/                # Business-ready
    └── sales_summary/
```

---

## IAM Policy Pattern

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/data/*"
    }
  ]
}
```

**Rules:**
- Never use `*` for Resource in production
- Separate roles per service
- Use conditions for extra security

---

## Glue Job Bookmarks

```python
# Enable in job parameters
--job-bookmark-option: job-bookmark-enable

# In your script
from awsglue.context import GlueContext
glueContext = GlueContext(SparkContext.getOrCreate())

# Read with bookmark tracking
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="mydb",
    table_name="orders",
    transformation_ctx="datasource"  # Required for bookmarks
)
```

---

## Athena Optimization

```sql
-- Partition pruning (10x cheaper)
SELECT * FROM orders
WHERE year = 2024 AND month = 1;  -- Uses partition columns

-- Projection pushdown (read fewer columns)
SELECT order_id, amount FROM orders;  -- Not SELECT *
```

---

## CloudFormation S3 Bucket

```yaml
Resources:
  DataLakeBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-datalake'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
```
