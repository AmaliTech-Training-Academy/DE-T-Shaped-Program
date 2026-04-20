# SK-04 Capstone Evaluation Rubric

## Grading Scale

| Level | Score | Description |
|-------|-------|-------------|
| **Exceeds** | 4 | Demonstrates mastery beyond requirements |
| **Meets** | 3 | Fully satisfies requirements |
| **Approaching** | 2 | Partial completion, minor gaps |
| **Below** | 1 | Significant gaps or missing |

**Pass threshold:** Average score of 3.0 or higher, no category below 2.

---

## Evaluation Criteria

### 1. S3 Data Lake Architecture (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Zone structure | Raw/processed/curated zones with clear purpose | Zones implemented | Partial structure | No zoning |
| Partitioning | Partitions match query patterns | Partitioning correct | Some partition issues | No partitioning |
| File formats | Parquet for analytics, appropriate compression | Formats correct | Format issues | Wrong formats |
| Naming conventions | Consistent, documented naming standards | Names consistent | Some inconsistency | No standards |

### 2. IAM & Security (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Least privilege | Roles have minimum required permissions | Permissions minimal | Some overpermission | Overly broad |
| Service roles | Separate roles for each service | Roles separated | Some shared roles | Single role |
| Encryption | SSE-S3 or KMS configured | Encryption enabled | Partial encryption | No encryption |
| Documentation | All policies documented with justification | Docs complete | Partial docs | No documentation |

### 3. AWS Glue ETL (25%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Job implementation | Multiple jobs with proper dependencies | Jobs work correctly | Minor issues | Jobs broken |
| Job bookmarks | Incremental processing with bookmarks | Bookmarks enabled | Bookmark issues | No bookmarks |
| Error handling | Dead-letter output for failed records | Errors handled | Partial handling | Pipeline crashes |
| Glue Catalog | Tables registered with proper schema | Catalog complete | Partial catalog | No catalog |

### 4. Athena Queries (15%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Query efficiency | Partition pruning, projection pushdown | Queries optimized | Some optimization | No optimization |
| Cost awareness | Data scanned minimized with partitions | Cost-conscious | Some waste | High scan costs |
| Documentation | Queries documented with purpose | Docs present | Partial docs | No documentation |

### 5. Infrastructure as Code (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| CloudFormation | All resources defined in templates | IaC complete | Partial IaC | No IaC |
| Parameterization | Environment-specific parameters | Params used | Hardcoded values | No parameters |
| Documentation | README with deployment instructions | Docs complete | Partial docs | No documentation |

---

## Submission Checklist

- [ ] `cloudformation/` - Infrastructure templates
- [ ] `glue_jobs/` - ETL job scripts
- [ ] `athena_queries/` - Sample queries
- [ ] `iam_policies/` - Policy documents with explanations
- [ ] `architecture_diagram.png` - Visual architecture
- [ ] `README.md` - Deployment and run instructions

---

## Common Failure Points

1. **Overly permissive IAM** - No `*` resources in production
2. **No partition pruning** - Always filter on partition columns
3. **Missing encryption** - All S3 buckets must be encrypted
4. **Console-only setup** - Infrastructure must be in CloudFormation
5. **No job bookmarks** - Full reload is not acceptable
