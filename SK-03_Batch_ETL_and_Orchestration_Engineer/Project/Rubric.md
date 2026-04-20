# SK-03 Capstone Evaluation Rubric

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

### 1. PySpark ETL Jobs (25%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Transformations | Complex transformations using proper PySpark patterns | All transforms work | Minor issues | Significant bugs |
| Schema handling | Explicit schemas, no schema inference in production | Schemas defined | Some inference used | No schema control |
| Partitioning | Output partitioned appropriately for downstream use | Partitioning correct | Partition issues | No partitioning |
| Performance | No unnecessary collect(), broadcast joins where appropriate | Efficient code | Some inefficiencies | Very inefficient |

### 2. Airflow DAG (25%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Task structure | Proper task dependencies with no circular refs | DAG runs correctly | Minor dependency issues | DAG broken |
| Idempotency | Tasks can be re-run safely without side effects | Tasks idempotent | Partial idempotency | Not idempotent |
| Error handling | Retries, alerts, and failure callbacks configured | Error handling present | Some error handling | No error handling |
| XComs | Data passed between tasks appropriately | XComs used correctly | XCom issues | No data passing |

### 3. dbt Models (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Model structure | Clear staging → intermediate → mart structure | Models organized | Some organization | Disorganized |
| Testing | Schema tests + custom tests on key models | Tests pass | Some tests | No tests |
| Documentation | dbt docs with descriptions and lineage | Docs complete | Partial docs | No documentation |
| Incremental | At least one incremental model | Incremental works | Incremental partial | No incremental |

### 4. Pipeline Patterns (15%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Incremental loading | High-water mark or CDC pattern implemented | Incremental works | Partial implementation | Full reload only |
| Data quality | Quality checks integrated into pipeline | Checks present | Some checks | No quality checks |
| Logging | Comprehensive logging with row counts | Logging complete | Basic logging | No logging |

### 5. Code Quality & Documentation (15%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| README | Complete setup, run, and troubleshooting guide | README adequate | Partial README | No README |
| Code organization | Modular, reusable functions | Well organized | Some organization | Monolithic |
| Reproducibility | Can run end-to-end with provided instructions | Runs correctly | Setup issues | Cannot reproduce |

---

## Submission Checklist

- [ ] `spark/` - PySpark job files
- [ ] `dags/` - Airflow DAG definitions
- [ ] `dbt_project/` - dbt models, tests, and docs
- [ ] `data/` - Sample input data (or generation script)
- [ ] `README.md` - Complete setup and run instructions
- [ ] `architecture.md` - Pipeline architecture diagram

---

## Common Failure Points

1. **collect() on large DataFrames** - Never collect production data
2. **Circular DAG dependencies** - Check with `airflow dags test`
3. **dbt tests not in CI** - Tests must run automatically
4. **No incremental pattern** - Full reload is not production-ready
5. **Missing Spark UI screenshots** - Include for performance analysis
