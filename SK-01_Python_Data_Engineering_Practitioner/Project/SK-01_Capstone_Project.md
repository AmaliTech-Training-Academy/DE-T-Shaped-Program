# SK-01 CAPSTONE PROJECT: Multi-Source HR Data Integration Pipeline for GlobalTech Corp

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **GlobalTech Corp**, a technology company that has just completed the acquisition of **AcquiredCo**, a smaller software firm. HR leadership needs a unified employee dataset delivered in **10 business days** to support:

- Day 1 integration planning (who is joining, in which role, at what cost)
- Benefits enrollment eligibility verification
- Payroll system migration to the combined company's platform
- Compliance reporting for the merger (headcount by jurisdiction, salary band distribution)

Data comes from 4 source systems that have never been integrated:

| Source | System | Format | Volume | Known Issues |
|---|---|---|---|---|
| GlobalTech HRIS | Workday export | CSV (UTF-8) | 15,000 employees | Department codes vary by business unit |
| AcquiredCo HRIS | BambooHR API | JSON (paginated) | 3,000 employees | Employee IDs overlap with GlobalTech's number range |
| Combined Payroll | ADP export | Excel (.xlsx) | 18,500 records | Salary in mixed currencies (USD, EUR, GBP); some records duplicated |
| Benefits Provider | MedShield XML export | XML | 12,000 records | Covers GlobalTech employees only; not all are enrolled |

### Why This Project Matters

| Risk if Data is Wrong | Business Impact |
|---|---|
| Duplicate employee records | Overpayment; incorrect headcount reported to regulators |
| Currency not converted | Salary bands and compensation analysis are invalid |
| Missing benefits enrollment | Employees miss open enrollment window; legal liability |
| Wrong jurisdiction codes | Compliance reports filed incorrectly; potential fines |

---

## PROJECT REQUIREMENTS

---

### Deliverable 1: Multi-Source Ingestion Module

Build a Python module (`ingest.py`) that loads all 4 sources into standardized Pandas DataFrames.

**Minimum requirements:**
- Separate ingestion function for each source with explicit parameter documentation
- XML ingestion using Python's built-in `xml.etree.ElementTree` or `lxml`
- Simulated API pagination for the AcquiredCo JSON source (read from file but implement as if paginating)
- Schema alignment function that maps all 4 sources to a standard employee schema (document every mapping decision)
- Source tagging and record counts logged at each ingestion step
- Graceful handling of missing files or malformed records (dead-letter logging, not pipeline crash)

---

### Deliverable 2: Data Cleaning & Transformation Module

Build a Python module (`clean.py`) with cleaning functions for each major problem class.

**Minimum requirements:**

**Name standardization:**
- Unicode normalization for accented characters
- Title case standardization
- Handling of hyphenated and multi-word last names ("Van Der Berg", "O'Brien")

**Employee ID resolution:**
- AcquiredCo IDs overlap with GlobalTech IDs (e.g., both have employee #1042)
- Build a namespaced ID system: `GT-001042` for GlobalTech, `AC-001042` for AcquiredCo

**Currency normalization:**
- Convert all salaries to USD using a fixed exchange rate table (provide your own rates)
- Handle salary strings: `"$85,000"` → `85000.0`
- Normalize pay frequency to annual: Monthly × 12, Bi-Weekly × 26
- Add `salary_usd_annual` as a new column; retain the original columns

**Department taxonomy mapping:**
- GlobalTech uses codes (`ENG-01`, `MKT-03`); AcquiredCo uses names (`Engineering`, `Marketing`)
- Build a mapping table that unifies both to a standard department taxonomy
- Log unmapped departments for manual review

**Date standardization:**
- GlobalTech: `YYYY-MM-DD`; AcquiredCo: `MM/DD/YYYY`; Benefits: `DD-Mon-YYYY` (e.g., `15-Jan-2022`)
- Normalize all to `datetime64[ns]`; flag dates outside plausible range (before 1970, after today)

---

### Deliverable 3: Deduplication Module

Build a Python module (`dedup.py`) with multi-pass deduplication logic.

**Minimum requirements:**

**Pass 1 — Exact employee ID match (within company namespace):**
- Records with the same `GT-XXXXXX` ID are duplicates of the same GlobalTech employee
- Apply source priority: HRIS > Payroll > Benefits

**Pass 2 — Email match (cross-company):**
- Same email address across GlobalTech and AcquiredCo = same person (rare but possible for contractors)

**Pass 3 — Fuzzy name + hire date match:**
- Use `rapidfuzz` to match first+last name combinations with similarity ≥ 88%
- Only consider records where hire date is within 30 days of each other (block by hire date proximity to avoid O(n²))
- Flag as `probable_match` rather than auto-merging — produce a review file for HR

**Ghost employee detection:**
- Records in Payroll with no corresponding HRIS record → flag as `ghost_employee = True` in a separate output file
- This is a compliance and fraud risk; must be separate from the main clean dataset

**Provenance tracking:**
- Every output record must have a `source_systems` column (e.g., `"globaltech_hris,payroll"`)
- Add `dedup_method` column (`exact_id`, `email_match`, `fuzzy_name`, `single_source`)

---

### Deliverable 4: Data Quality Validation Report

Build a validation module (`validate.py`) using the `DataQualityValidator` class pattern from the guided lab (or Great Expectations if you choose).

**Minimum requirements — at least 12 checks:**

| Check Type | Fields to Check |
|---|---|
| NOT NULL | `employee_id`, `first_name`, `last_name`, `email`, `department`, `country` |
| UNIQUE | `email` (post-dedup), `employee_id` (post-dedup) |
| VALUES IN SET | `employment_type` ∈ {Full-Time, Part-Time, Contractor}, `currency` ∈ {USD, EUR, GBP} |
| REGEX | `email` format, `employee_id` format (must match `GT-\d{6}` or `AC-\d{6}`) |
| NUMERIC RANGE | `salary_usd_annual` between $15,000 and $2,000,000 |
| DATE RANGE | `hire_date` between 1970-01-01 and today |
| REFERENTIAL INTEGRITY | Every `manager_id` that is not null must exist as an `employee_id` in the dataset |

**Report format:**
- DataFrame with columns: `check`, `description`, `total`, `passed`, `failed`, `pass_rate`, `status`
- Overall pipeline gate: if more than 2 checks fail, halt and log a critical error
- Export as both CSV and an HTML summary table

---

### Deliverable 5: EDA & Visualization Report

Produce a professional visualization report saved as a high-resolution PNG (300 DPI minimum).

**Minimum 6 charts:**

| # | Chart | Purpose |
|---|---|---|
| 1 | Headcount by Department (horizontal bar) | Show combined org structure |
| 2 | Headcount by Country (choropleth or bar) | Show geographic distribution |
| 3 | Salary Distribution by Employment Type (violin or box) | Compensation analysis |
| 4 | Tenure Distribution (histogram) | Workforce experience profile |
| 5 | Benefits Enrollment Rate by Department (bar) | HR planning |
| 6 | Data Quality Summary (grouped bar: passed vs failed per check) | Pipeline health dashboard |

**Styling requirements:**
- Use a colorblind-safe palette
- Every chart must have a title, axis labels, and a data source annotation
- Figure must have an overall title and a generation timestamp

---

### Deliverable 6: Golden Dataset & Documentation

**Golden employee dataset (Parquet):**
- Unified, cleaned, deduplicated employee records
- Schema documented with column name, data type, description, and example value
- Partitioned by `company_origin` (GlobalTech vs AcquiredCo) for downstream filtering

**Ghost employee report (CSV):**
- All payroll records with no HRIS match
- Required fields: `payroll_employee_id`, `name`, `salary_usd_annual`, `ghost_flag_reason`

**Probable match review file (CSV):**
- Fuzzy-matched pairs for HR to confirm or reject
- Required fields: `record_1_id`, `record_2_id`, `similarity_score`, `hire_date_diff_days`, `recommended_action`

**README.md:**
- Pipeline purpose and business context
- Input sources (file paths, formats, expected schemas)
- Output files (paths, formats, descriptions)
- How to run the pipeline
- Known limitations and assumptions
- Change log

