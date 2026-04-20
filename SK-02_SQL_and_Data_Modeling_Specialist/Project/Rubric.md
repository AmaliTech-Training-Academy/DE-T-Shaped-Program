# SK-02 Capstone Evaluation Rubric

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

### 1. Dimensional Model Design (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Grain definition | Clear, unambiguous grain statement for both facts | Both grains defined | One grain unclear | Missing grain |
| SCD decisions | All dimensions have justified SCD type | SCD types correct | Some justifications missing | Incorrect SCD types |
| Fact classification | Transaction/periodic/accumulating properly identified | Classification correct | Classification unclear | Not classified |
| Conformed dimensions | At least one dimension designed for reuse | Conformed dim present | Conformed dim partial | No conformed dims |

### 2. Physical DDL (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Schema completeness | All dimensions and facts with proper types | All tables present | Minor type issues | Missing tables |
| Surrogate keys | All dimensions use SERIAL/BIGSERIAL PKs | Surrogates correct | Some natural keys | No surrogates |
| SCD columns | effective_start, effective_end, is_current present | SCD columns correct | Some columns missing | No SCD columns |
| Indexes | All FK indexes created with justification | Indexes appropriate | Some indexes missing | No indexes |

### 3. ETL SQL (25%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| SCD Type 2 | Close-then-insert pattern with IS DISTINCT FROM | SCD2 works correctly | SCD2 has minor bugs | SCD2 broken |
| dim_date | Pure SQL generation covering full range | Date dimension complete | Date dim partial | Missing date dim |
| Fact loading | Incremental with SCD-aware lookups | Fact loads correctly | Some lookup issues | Facts don't load |
| Dead letter | Unmatched records logged, not failed | DLQ implemented | Partial error handling | Pipeline crashes |

### 4. Analytical Queries (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Query count | All 8 queries present and correct | 8 queries work | 6-7 queries work | Fewer than 6 |
| Window functions | Proper use of RANK, LAG, SUM OVER | Windows used correctly | Some window issues | No window functions |
| Documentation | Each query explains business purpose | All queries documented | Some docs missing | No documentation |
| Performance | Queries return in reasonable time | Queries perform OK | Some slow queries | Very slow queries |

### 5. Performance Optimization (15%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Index strategy | Complete index list with justifications | Indexes documented | Partial documentation | No index strategy |
| EXPLAIN analysis | Clear interpretation of execution plans | 2 plans analyzed | 1 plan analyzed | No analysis |
| Materialized view | MV created with refresh recommendation | MV implemented | MV partial | No MV |
| Before/after | Performance comparison documented | Comparison shown | Partial comparison | No comparison |

---

## Submission Checklist

- [ ] `design_document.md` - Dimensional model design
- [ ] `ddl/` - All CREATE TABLE statements
- [ ] `etl/` - All ETL SQL scripts
- [ ] `queries/` - 8 analytical queries with comments
- [ ] `optimization_report.md` - Performance analysis
- [ ] `README.md` - Setup and run instructions

---

## Common Failure Points

1. **Natural keys instead of surrogates** - All dimensions need SERIAL PKs
2. **SCD2 verification query missing** - Must prove no duplicate is_current=TRUE
3. **Hardcoded dates** - dim_date should use GENERATE_SERIES
4. **No IS DISTINCT FROM** - NULL comparison will fail silently
5. **Missing EXPLAIN output** - Include actual plan, not just commentary
