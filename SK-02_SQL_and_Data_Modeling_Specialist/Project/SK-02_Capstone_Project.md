# SK-02 CAPSTONE PROJECT: Healthcare Claims Data Warehouse for MedInsure

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **MedInsure**, a regional health insurance provider processing **2 million claims per year**. The Claims Analytics team currently runs all reporting directly against the OLTP claims processing database — a 3NF schema that was designed for transaction speed, not analysis.

The situation is unsustainable:

- The **Provider Performance Report** (required for monthly contract reviews) takes 6 hours to run
- The **Member Utilization Dashboard** crashes the OLTP database twice a week under query load
- Finance cannot answer the question "what did we pay per diagnosis category last quarter?" without 3 days of manual work
- The compliance team requires a **3-year history** of member plan changes for regulatory audits — the OLTP system only retains 18 months

The Chief Analytics Officer has approved a dedicated analytics data warehouse. Your job: design the complete dimensional model, implement SCD Type 2, write all ETL SQL, deliver analytical queries that answer the business questions above, and optimize performance.

---

## SOURCE DATA DESCRIPTION

The following tables exist in the OLTP claims processing system. You must generate realistic synthetic data for all of them.

| Table | Description | Approx. Volume |
|---|---|---|
| `members` | Insured individuals with demographics, plan, and enrollment data | 500,000 |
| `member_plan_history` | Historical plan changes (used to build SCD) | 750,000 rows |
| `providers` | Doctors, hospitals, and facilities with specialty and network status | 15,000 |
| `provider_network_history` | Historical in/out-of-network status changes | 22,000 rows |
| `claims` | One claim per visit/encounter (header level) | 2,000,000 |
| `claim_lines` | Individual service lines within a claim | 7,500,000 |
| `diagnosis_codes` | ICD-10 reference (code + description + category) | 12,000 |
| `procedure_codes` | CPT reference (code + description + category) | 8,500 |
| `plan_types` | Insurance plan reference (HMO, PPO, EPO, HDHP) | 8 |

---

## PROJECT REQUIREMENTS

### Deliverable 1: Dimensional Model Design

Produce a written design document (Markdown) before writing any DDL.

**Minimum requirements:**
- Grain declaration for both fact tables (state it in one sentence each)
- Dimension list with SCD type decision and justification for each
- Fact table type classification (transaction / periodic snapshot / accumulating snapshot) with justification
- At least one conformed dimension that would be shared with a future `fact_pharmacy` table
- Identification of any degenerate dimensions

**Fact tables to design:**
- `fact_claim` — header-level grain (one row per claim)
- `fact_claim_line` — line-level grain (one row per service line within a claim)

---

### Deliverable 2: Physical DDL

Write complete CREATE TABLE statements for all dimensions and both fact tables.

**Minimum dimension tables:**
- `dim_date` (use the GENERATE_SERIES pattern from the lab — customize for calendar year, no fiscal calendar needed)
- `dim_member` (SCD Type 2 — tracks plan changes)
- `dim_provider` (SCD Type 2 — tracks network status changes)
- `dim_diagnosis` (SCD Type 1 — ICD-10 codes with category hierarchy, codes rarely change)
- `dim_procedure` (SCD Type 1 — CPT codes with category hierarchy)
- `dim_plan` (SCD Type 1 — plan attributes: deductible, OOP max, plan type)

**Required DDL standards:**
- Every dimension must have a surrogate primary key (SERIAL or BIGSERIAL)
- Every SCD Type 2 dimension must have `effective_start`, `effective_end`, `is_current`
- Every fact table FK must have an index
- Include a "Not Applicable" placeholder row for any nullable dimension FK
- All columns must have appropriate data types (no VARCHAR(255) for everything)
- Add inline comments explaining non-obvious design decisions

---

### Deliverable 3: ETL SQL

Write complete ETL SQL to populate all dimensions and both fact tables.

**Minimum requirements:**

**dim_date:** Full GENERATE_SERIES population for 2020–2030 (no Python, pure SQL)

**dim_member (SCD Type 2):**
- Use the two-step close-then-insert pattern from the lab
- Change detection must use IS DISTINCT FROM for NULL safety
- Track changes to: `plan_id`, `state`, `zip_code`
- Include a verification query that confirms no member has more than one is_current=TRUE row

**dim_provider (SCD Type 2):**
- Track changes to: `network_status`, `specialty`
- Include the same verification query

**dim_diagnosis and dim_procedure (Type 1):**
- Simple UPSERT (INSERT ... ON CONFLICT DO UPDATE)

**fact_claim (header load):**
- Incremental load using high-water mark on `processed_date`
- SCD-aware member and provider key lookups using date-range join
- Include handling for members or providers not found in the dimension (write to a dead-letter log table, do not fail the pipeline)

**fact_claim_line (line load):**
- Load after `fact_claim` (dependency ordering)
- Look up `fact_claim.claim_key` using the degenerate `claim_id` on the header fact

---

### Deliverable 4: Analytical Queries

Write the following 8 analytical queries. Each must run on the warehouse schema (not the source OLTP), and each must include a comment explaining what business question it answers and why the warehouse schema makes it possible.

| # | Query | Window Functions Required? |
|---|---|---|
| 1 | Top 20 providers by total paid amount, with YoY comparison | Yes — LAG or self-join |
| 2 | Claims paid by ICD-10 diagnosis category, ranked by spend | Yes — RANK() |
| 3 | Member utilization rate by plan type and month (claims per 1,000 members) | Yes — COUNT OVER |
| 4 | Denied claims analysis: denial rate by provider specialty and claim type | No — GROUP BY |
| 5 | Out-of-network cost comparison: paid amount per member per year (in-network vs out-of-network) | No — conditional aggregation with CASE |
| 6 | Monthly claims volume trend with 3-month moving average | Yes — AVG() OVER ROWS |
| 7 | Member cohort analysis: claims frequency in first 12 months after enrollment by enrollment year | Yes — complex CTE + window |
| 8 | Running cumulative paid amount by provider YTD, flagging when a provider crosses $1M | Yes — SUM() OVER cumulative |

---

### Deliverable 5: Performance Optimization

Deliver a written optimization report (Markdown) covering:

**Index strategy:**
- List every index you created on both fact tables
- For each index, specify: columns, index type (B-tree/covering), and the query it supports
- Identify any index you chose NOT to create and why (write overhead vs benefit trade-off)

**Execution plan analysis:**
- Run EXPLAIN ANALYZE on queries #1 and #6 from Deliverable 5
- Include the full EXPLAIN output as a code block
- Write a 3–5 sentence plain English interpretation of each plan
- Identify the most expensive node in each plan and explain what it is doing

**Materialized view:**
- Create one materialized view for the monthly claims summary (query #6)
- Show the EXPLAIN output before (base tables) and after (materialized view)
- Write a refresh schedule recommendation and justify it

---

### Deliverable 6: Data Quality Report

Run quality checks on the loaded warehouse and produce a report.

**Minimum checks:**
- `fact_claim`: Every claim has a valid `date_key`, `member_key`, `provider_key`
- `fact_claim`: No claims where `total_paid > total_allowed * 1.05` (5% tolerance for rounding)
- `dim_member`: No member has more than one `is_current = TRUE` row
- `dim_provider`: No provider has more than one `is_current = TRUE` row
- Referential integrity: Every `claim_id` in `fact_claim_line` exists in `fact_claim`
- Orphaned procedure codes: Report count of claim lines with procedure codes not in `dim_procedure`
- Providers with anachronistic claims: Claims where `service_date < provider.network_effective_date`

Present results as a formatted table: check name, total rows evaluated, passed, failed, pass rate, status (PASS/FAIL at 99% threshold).

---

