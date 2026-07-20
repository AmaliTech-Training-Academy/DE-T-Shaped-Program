# SK-06 CAPSTONE PROJECT: Multi-Tenant SaaS Analytics Dashboard for CloudHR

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **CloudHR**, a SaaS company providing HR management software to **200 client organisations** — ranging from 50-person startups to 10,000-employee enterprises. Each client organisation has an HR team that needs self-service analytics covering workforce metrics, compensation, leave patterns, and performance trends.

**The critical constraint:** Each client organisation must see **only their own data**. A user at Company A must never be able to see Company B's headcount, salaries, or employee details — even if they discover how to manipulate the URL or share a report link.

**Current situation:** CloudHR's customer success team manually runs SQL queries for each client every month and sends them an Excel spreadsheet. At 200 clients, this takes 3 analysts 2 weeks per month. Clients complain that the data is always 3–4 weeks behind.

**Target:** A single Power BI solution that serves all 200 clients simultaneously, with each client seeing only their own data, refreshing weekly, and accessible on mobile devices.

---

## SOURCE DATA DESCRIPTION

The CloudHR backend database exposes the following tables. You must generate realistic synthetic data for all of them.

| Table | Description | Key Columns |
|---|---|---|
| `organisations` | The 200 client companies | org_id, org_name, industry, employee_count_band, country, subscription_tier |
| `employees` | Current and historical employees (SCD Type 2) | employee_id, org_id, first_name, last_name, department_id, job_level, employment_type, hire_date, termination_date, termination_reason |
| `departments` | Department hierarchy | department_id, org_id, department_name, department_group, cost_centre |
| `employee_monthly_snapshots` | One row per employee per month (current state at month end) | snapshot_month, employee_id, org_id, is_active, base_salary_usd, job_level, department_id |
| `leave_records` | Individual leave instances | leave_id, employee_id, org_id, leave_type, start_date, end_date, days_taken, approved_by |
| `performance_reviews` | Annual performance ratings | review_id, employee_id, org_id, review_year, rating (1-5), reviewer_id |
| `security_mapping` | Maps user emails to their org_id | user_email, org_id, access_level (HR\_ADMIN, HR\_VIEWER) |

---

## PROJECT REQUIREMENTS

---

### Deliverable 1: Data Model Design Document

Before building anything in Power BI, produce a written design document (Markdown).

**Required decisions with justifications:**

**Schema design:** Should `employee_monthly_snapshots` be the fact table, or should `employees` be the fact table? Justify your choice in terms of the grain, the measures you need, and how headcount should be calculated.

**Semi-additive handling:** Why is headcount semi-additive? What would go wrong if you used a simple SUM for headcount when a user selects "Q1 2025" in a slicer?

**Relationship design:** How do you handle the fact that `leave_records` and `performance_reviews` link to `employees` (not directly to `employee_monthly_snapshots`)? Draw the star schema.

**Date table:** What fiscal year start month will you use, and why? (Hint: ask what makes sense for an HR system — most HR metrics reset at the calendar year, but bonus cycles may differ.)

**Many-to-many:** One department can contain employees from multiple job levels. One job level contains employees across multiple departments. Is this a many-to-many in the data model? How do you handle it?

---

### Deliverable 2: Power Query Transformations

Write complete M code (or GUI-built steps documented with the equivalent M) for:

**`fact_employee_monthly`** (the primary fact table):
- Source: `employee_monthly_snapshots` joined with `employees`
- Add: `DateKey` (integer YYYYMMDD for the last day of the snapshot month)
- Handle: salary records with zero or negative values → null and log to a separate error table
- Handle: employees with hire_date after termination_date → add an `is_data_error` flag
- Remove: all PII columns not needed for analytics (first_name, last_name — use employee_id only)
- Set: all explicit data types

**`dim_date`**:
- Calendar year (January 1 start — standard for HR metrics)
- Columns: DateKey, Date, Year, Month Number, Month Name, Quarter, Day of Week, Is Weekend, Is Holiday (US Federal), Week Number
- Sort-by-column configuration: Month Name sorted by Month Number

**`dim_employee`** (Type 2 history — current snapshot only):
- Only `is_active = TRUE` and `termination_date IS NULL` rows
- This is the dimension for browsing current employees; `fact_employee_monthly` contains the history

**`security_mapping`**:
- Hard-coded M table for the lab (replace with SQL source in production)
- Include at least 3 test users: one HR\_ADMIN, two HR\_VIEWER from different organisations

---

### Deliverable 3: DAX Measure Library (15+ Measures)

Write and verify at least 15 DAX measures. For each measure, include a comment explaining the business definition and any non-obvious implementation choices.

**Required measures:**

| # | Measure Name | Pattern | Why Non-Trivial |
|---|---|---|---|
| 1 | Active Headcount | Semi-additive (LASTDATE) | Cannot SUM across months |
| 2 | Headcount EOP (End of Period) | Semi-additive variant | Show count at period end |
| 3 | Headcount BOP (Beginning of Period) | Semi-additive FIRSTDATE | For turnover denominator |
| 4 | New Hires | CALCULATE + date filter | Hires in the selected period only |
| 5 | Terminations | CALCULATE + termination_date filter | Separations in selected period |
| 6 | Turnover Rate | (Terminations / Avg Headcount) × 100 | Industry-standard formula |
| 7 | Voluntary Turnover Rate | Filter by termination_reason | Subset of turnover |
| 8 | Avg Tenure (Years) | AVERAGEX over active employees | Requires date arithmetic |
| 9 | Avg Base Salary | AVERAGEX with null exclusion | Exclude $0 salaries |
| 10 | Compensation Band Width | MAX(salary) − MIN(salary) | Pay equity analysis |
| 11 | Headcount YoY Growth % | SAMEPERIODLASTYEAR | Time intelligence |
| 12 | Headcount YTD | Semi-additive YTD (LASTDATE + DATESYTD) | Combine both patterns |
| 13 | Leave Utilization % | Days taken / entitlement days | Optional join required |
| 14 | Performance Rating Avg | AVERAGE with period filter | Simple but verify correctly |
| 15 | % Workforce at Each Job Level | DIVIDE with ALLEXCEPT | Distribution within context |

---

### Deliverable 4: Report — 4 Pages

**Page 1: Executive Workforce Summary**

Required elements:
- 4 KPI cards: Active Headcount, Turnover Rate, Avg Tenure, Avg Base Salary
- Headcount trend line chart (current period vs prior year)
- Headcount by Department Group bar chart (horizontal)
- New Hires vs Terminations clustered bar chart by month
- Slicers: Year, Quarter, Department Group, Employment Type

**Page 2: Workforce Composition**

Required elements:
- Headcount distribution by Job Level (stacked bar — shows seniority pyramid)
- Headcount by Employment Type (donut chart)
- Average Tenure by Department (bar chart — sorted descending)
- A table: Top 10 Departments by Headcount with YoY Growth % and Turnover Rate

**Page 3: Compensation Analytics**

Required elements:
- Average Salary by Job Level (bar chart — essential for pay equity review)
- Salary Distribution (histogram — shows spread within the organisation)
- Compensation Band Width by Department (are salaries compressed or spread?)
- A scatter plot: Tenure vs Salary (do longer-tenured employees earn more?)
- Filter: Department, Job Level, Employment Type

**Page 4: Leave & Attendance** *(use leave_records as the source)*

Required elements:
- Leave Utilization % by Leave Type (horizontal bar)
- Monthly Leave Days Trend (line chart with 3-month moving average)
- Top 10 Departments by Leave Days (who is taking the most time off?)
- Leave Type Breakdown (donut: Annual, Sick, Parental, Other)

**Design requirements for all pages:**
- Consistent colour theme (not Power BI default)
- Drill-through: clicking a department on any page drills through to a department detail page (you design this 5th page)
- Back button on the detail page
- Mobile layout configured on at least Page 1

---

### Deliverable 5: Multi-Tenant RLS

**Implement and document the following security model:**

**Role: "Tenant Access"** — the single role used for all users

Dynamic RLS filter on `dim_organisation` table:
```dax
[org_id] = LOOKUPVALUE(
    security_mapping[org_id],
    security_mapping[user_email],
    USERPRINCIPALNAME()
)
```

**Testing requirements:**
- Test with 3 different user emails (using "View as → Other user" in Desktop)
- User from Org A: verify they see Org A headcount, not Org B or C
- **Critical negative test:** document in your submission that you verified a user from Org B cannot see Org A's average salary
- Test with a user email NOT in the security mapping — what happens? What should happen?

**Documentation:**
- Write a 1-page RLS Administration Guide: how to add a new user, how to revoke access, how to test access after changes, what to do if a user reports seeing wrong data

---

### Deliverable 6: Performance Report

Run Performance Analyzer on all 4 report pages and document findings.

**Required content:**

- Screenshot or table of Performance Analyzer results for every visual on all 4 pages
- Identify the slowest visual (most likely a complex DAX measure)
- Paste the DAX for that measure into DAX Studio with Server Timings
- Document: FE (formula engine) time, SE (storage engine) time, number of SE queries
- Apply at least one optimization (use VAR to eliminate a redundant sub-expression, or remove an unused column, or change a calculated column to a measure)
- Document before/after Performance Analyzer times

**Target:** All visuals must be < 800ms DAX query time. At least 80% must be < 400ms.

---

### Deliverable 7: Deployment Guide

Write a Markdown deployment guide covering:

**Workspace structure:**
- Which workspaces do you create? (Dev, Test, Prod — or different names?)
- Who has what permissions in each workspace?

**Deployment pipeline:**
- How does a change flow from development to production?
- What is the approval gate before production deployment?
- How do you configure deployment rules to point Dev vs Prod at different database connections?

**Refresh configuration:**
- What schedule do you recommend? (Weekly on Sunday night at 2 AM?)
- What gateway configuration is needed?
- Who gets notified on refresh failure?

**Onboarding a new client organisation:**
- What is the exact sequence of steps to give a new 201st client organisation access to their own data?
- How long should onboarding take end-to-end?
