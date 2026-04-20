# SK-01 Capstone Evaluation Rubric

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

### 1. Multi-Source Ingestion (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Format handling | Handles CSV, JSON, XML, and fixed-width with encoding detection | All 4 formats work correctly | 3 of 4 formats work | Fewer than 3 formats |
| Schema alignment | Standardizes all sources to unified schema with type casting | Schema alignment complete | Minor type issues | Schema inconsistent |
| Error handling | Graceful failure with dead-letter logging | Errors logged, no crashes | Some error handling | Crashes on bad data |
| Logging | Informative logs with row counts at each step | Adequate logging | Minimal logging | No logging |

### 2. Data Cleaning & Transformation (25%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Name standardization | Unicode normalization + title case + edge cases | Consistent standardization | Most names cleaned | Inconsistent |
| ID namespacing | Clear namespace system preventing collisions | IDs properly namespaced | Minor issues | ID collisions exist |
| Currency handling | Conversion + frequency normalization + original retained | All conversions correct | Some currency issues | Missing conversion |
| Date standardization | All formats handled with validation | Dates normalized | Some formats fail | Dates inconsistent |

### 3. Deduplication (20%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Multi-pass logic | Exact + fuzzy matching with proper blocking | Both passes work | Only exact matching | No deduplication |
| Source priority | Clear priority with field-level merging | Priority respected | Priority partially works | No priority logic |
| Ghost detection | Ghost records identified and exported separately | Ghost detection works | Partial detection | Not implemented |
| Provenance | Full source tracking on every record | Sources tracked | Some tracking | No provenance |

### 4. Data Quality Validation (15%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Check coverage | 12+ checks across all dimensions | All required checks pass | Most checks implemented | Fewer than 10 checks |
| Report format | CSV + HTML with clear pass/fail | Both formats produced | One format only | No report |
| Pipeline gate | Fails pipeline if threshold not met | Gate implemented | Gate partially works | No gate |

### 5. Visualization (10%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Chart quality | 6+ professional charts with colorblind-safe palette | All 6 charts present | 4-5 charts | Fewer than 4 |
| Labels | All charts have title, axis labels, data source | Labels complete | Some labels missing | Poor labeling |
| Insight clarity | Charts immediately convey data quality story | Charts are clear | Some charts unclear | Confusing charts |

### 6. Code Quality (10%)

| Criteria | Exceeds (4) | Meets (3) | Approaching (2) | Below (1) |
|----------|-------------|-----------|-----------------|-----------|
| Structure | Modular functions, clear separation of concerns | Well organized | Some organization | Monolithic code |
| Documentation | README + inline comments explaining decisions | Adequate docs | Minimal docs | No documentation |
| Reproducibility | Can run end-to-end with clear instructions | Runs correctly | Some setup issues | Cannot reproduce |

---

## Submission Checklist

- [ ] `ingest.py` - Multi-source ingestion module
- [ ] `clean.py` - Cleaning and transformation module
- [ ] `dedup.py` - Deduplication module
- [ ] `validate.py` - Data quality validation module
- [ ] `golden_employees.parquet` - Final deduplicated dataset
- [ ] `ghost_employees.csv` - Records with no HRIS match
- [ ] `probable_matches.csv` - Fuzzy match review file
- [ ] `quality_report.csv` and `.html` - Validation results
- [ ] `eda_report.png` - Visualization output
- [ ] `README.md` - Setup and run instructions

---

## Common Failure Points

1. **Currency conversion missing** - All salaries must be in USD
2. **ID collisions not resolved** - Overlapping IDs must be namespaced
3. **Ghost employees not exported** - Must be a separate file
4. **No fuzzy matching** - Exact match alone is insufficient
5. **Tests don't run** - Include test instructions in README
