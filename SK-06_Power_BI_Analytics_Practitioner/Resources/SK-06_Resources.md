# SK-06 RESOURCES: Power BI & Analytics Practitioner

---

## How to Use This Resource Guide

Items marked **(Essential)** should be completed by all participants. Items marked **(Recommended)** deepen understanding. Items marked **(Advanced)** target participants pursuing project-ready mastery.

---

---

## MODULE A: Data Modeling in Power BI

---

### A.1–A.2: Architecture and Star Schema Design

#### Articles

1. **(Essential)** [Star Schema — The Foundation of Power BI (SQLBI)](https://www.sqlbi.com/articles/star-schema-the-foundation-of-power-bi/)
   - **Why:** SQLBI is the definitive source for Power BI best practices. This article explains exactly why star schema outperforms flat tables and snowflakes in Power BI — with benchmark data. Required reading before building any data model.
   - **Time:** 20 min read

2. **(Essential)** [Understand Star Schema and the Importance for Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema)
   - **Why:** The official Microsoft guidance on star schema design for Power BI. Covers fact tables, dimension tables, relationship patterns, and specific Power BI considerations like the role-playing dimension.
   - **Time:** 30 min read

3. **(Recommended)** [Import vs DirectQuery — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-directquery-about)
   - **Why:** The complete guide to DirectQuery: when to use it, what it can't do (some DAX functions don't work, time intelligence is limited), and performance implications. Essential for choosing the right storage mode on a project.
   - **Time:** 25 min read

4. **(Advanced)** [VertiPaq Analyzer — Understanding Compression](https://www.sqlbi.com/tools/vertipaq-analyzer/)
   - **Why:** Free tool from SQLBI that shows column-level compression, cardinality, and memory usage in your model. Tells you exactly which columns are consuming the most model size and why. Indispensable for model optimization on large datasets.
   - **Time:** 20 min to read documentation; use as ongoing diagnostic tool

#### Videos

1. **(Essential)** [Data Modeling in Power BI: Complete Guide — Guy in a Cube](https://www.youtube.com/watch?v=MrLnibFTtbA)
   - **Why:** Practical walkthrough of building a star schema from scratch in Power BI Desktop. Shows Model View, creating relationships, setting cardinality, and common mistakes.
   - **Duration:** 45 min

2. **(Recommended)** [Calculation Groups in Power BI — SQLBI](https://www.youtube.com/watch?v=nLTiKHsKGjc)
   - **Why:** Calculation groups eliminate the need to write separate time intelligence measures for every measure (YTD Revenue, YTD Cost, YTD Margin, YTD Orders...). This video is the clearest explanation of how they work and when to use Tabular Editor.
   - **Duration:** 35 min

#### Custom Resources

1. **(Essential)** **Star Schema Design Checklist**

   ```
   BEFORE BUILDING ANY POWER BI DATA MODEL:

   □ Define the grain of your fact table.
     Write it as one sentence: "One row per ___"
     Example: "One row per order line item per day"

   □ List all measures you need.
     Measures come from the fact table. If a measure can't be derived
     from the fact + dimensions, you may need a different fact table.

   □ List all dimension attributes.
     Every non-measure column that a user would filter or group by
     should be in a dimension table, not the fact table.

   □ Verify: fact table has no attributes — only measures and keys.
     The only text in the fact table should be degenerate dimensions
     (e.g., order_id — doesn't need its own dimension table).

   □ Check for snowflake structure:
     If a dimension table has a foreign key to another dimension table
     (e.g., dim_product → dim_category → dim_category_group),
     FLATTEN it in Power Query. Power BI works best with one-hop relationships.

   □ Use integer surrogate keys for all relationships.
     Not string GUIDs. Not natural keys (which may be strings).
     Integer keys = faster VertiPaq compression = smaller model.

   □ Verify relationship directions.
     ALL relationships in the star schema should filter FROM dimension TO fact.
     Single-direction (the default). Never bidirectional except for M:M bridge tables.

   □ Confirm no ambiguous filter paths.
     A fact table should be reachable from each dimension by exactly one path.
     Multiple paths = ambiguity error in Power BI.
   ```

---

### A.3–A.4: Relationships and Advanced Modeling

#### Articles

1. **(Essential)** [Bidirectional Relationships in Power BI — SQLBI](https://www.sqlbi.com/articles/bidirectional-relationships-and-ambiguity-in-dax/)
   - **Why:** Explains exactly why bidirectional relationships are dangerous (ambiguous filter paths, incorrect totals, performance issues) and the specific cases where they are correct (many-to-many bridge tables). Prevents the most common data modeling mistake.
   - **Time:** 20 min read

2. **(Recommended)** [Many-to-Many Relationships — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/guidance/relationships-many-to-many)
   - **Why:** Covers all three approaches to many-to-many: the limited M:M cardinality setting, the bridge table pattern, and TREATAS. Shows which approach is appropriate for each use case with examples.
   - **Time:** 25 min read

---

---

## MODULE B: Power Query (M) — Data Preparation

---

### B.1–B.2: Power Query and Query Folding

#### Articles

1. **(Essential)** [Query Folding in Power Query — Microsoft Docs](https://learn.microsoft.com/en-us/power-query/power-query-folding)
   - **Why:** The authoritative Microsoft explanation of query folding — what it is, which transformations fold, and how to check if folding is active. The "View Native Query" and "View Query Plan" sections are especially practical.
   - **Time:** 25 min read

2. **(Essential)** [Power Query M Reference — Microsoft Docs](https://learn.microsoft.com/en-us/powerquery-m/power-query-m-function-reference)
   - **Why:** The complete M function library. Bookmark this — use it when you need to find the exact M function for a text, date, or number operation. The Table, Text, Date, and List categories are the most frequently used.
   - **Time:** Reference — use as needed

3. **(Recommended)** [Best Practices for Power Query Performance — Microsoft Docs](https://learn.microsoft.com/en-us/power-query/best-practices)
   - **Why:** Official performance best practices: use native queries where possible, keep non-foldable steps at the end, disable loading queries that only support other queries (reference queries). Clear and actionable.
   - **Time:** 20 min read

4. **(Advanced)** [Query Diagnostics in Power Query](https://learn.microsoft.com/en-us/power-query/query-diagnostics)
   - **Why:** The Power Query Diagnostics pane shows exactly how long each step takes and whether it folded. Essential for identifying the step that broke folding and measuring the improvement after fixing it.
   - **Time:** 20 min read + hands-on practice

#### Videos

1. **(Essential)** [Power Query Full Tutorial — Pragmatic Works](https://www.youtube.com/watch?v=0aeZX1l4JT4)
   - **Why:** Comprehensive Power Query tutorial covering the editor UI, all core transformations, M code, and query folding. The folding section at around the 90-minute mark is especially clear.
   - **Duration:** 2 hours (watch at 1.5×)

2. **(Recommended)** [Power Query Query Folding Deep Dive — How to Power BI](https://www.youtube.com/watch?v=3HbsFUCPFYk)
   - **Why:** Detailed demonstration of enabling/breaking/restoring query folding. Shows the View Native Query output and the performance difference with real measurements.
   - **Duration:** 25 min

#### Custom Resources

1. **(Essential)** **Query Folding Decision Guide**

   ```
   TRANSFORMATIONS THAT FOLD TO SQL:
   ✓ Table.SelectRows        → WHERE clause
   ✓ Table.SelectColumns     → SELECT column list
   ✓ Table.TransformColumnTypes → CAST
   ✓ Table.RenameColumns     → column alias (AS)
   ✓ Table.NestedJoin        → JOIN
   ✓ Table.Group             → GROUP BY + aggregates
   ✓ Table.Sort              → ORDER BY
   ✓ Table.FirstN            → TOP N (SQL Server) / LIMIT N (Postgres)
   ✓ Table.AddColumn with simple arithmetic (x + y, x * y) → computed column
   ✓ Table.RemoveColumns     → narrower SELECT

   TRANSFORMATIONS THAT BREAK FOLDING:
   ✗ Custom M functions (any function you define with let...in)
   ✗ Table.Buffer()           (forces evaluation to break dependency)
   ✗ List.Contains()          in a column condition
   ✗ Complex conditional columns referencing multiple table lookups
   ✗ Text operations with no SQL equivalent (Text.BetweenDelimiters etc.)
   ✗ Combining tables with different sources (S3 merged with SQL)

   RECOVERY STRATEGIES:
   1. Move non-foldable steps to the END of the chain
      (after all filtering, joining, and grouping)
   2. Use a native SQL query as the source instead of GUI transformations
      (SQL Server: Sql.Database with a custom Query parameter)
   3. Apply a Table.Buffer() BEFORE the non-foldable step to checkpoint
      what has been folded so far (the buffered result is the new "source")

   DIAGNOSTIC:
   Right-click any step → "View Native Query"
   ✓ If native query shows → folding is active
   ✗ If grayed out → folding is broken at or before this step
   ```

---

### B.3–B.4: Date Dimension and Parameters

#### Articles

1. **(Essential)** [Create a Date Table in Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/guidance/model-date-tables)
   - **Why:** The official guidance on date table requirements: must be continuous (no gaps), must cover all dates in the fact table, must be marked as a date table. Also explains the auto date/time feature and why to disable it.
   - **Time:** 15 min read

2. **(Recommended)** [Disable Auto Date/Time — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/guidance/auto-date-time)
   - **Why:** Auto date/time creates hidden date hierarchies for every date column — including ones you don't want. This inflates model size significantly. This article explains what it creates and how to disable it.
   - **Time:** 10 min read

---

---

## MODULE C: DAX — Data Analysis Expressions

---

### C.1–C.2: DAX Fundamentals and CALCULATE

#### Articles

1. **(Essential)** [DAX Patterns — SQLBI](https://www.daxpatterns.com/)
   - **Why:** The single most important DAX reference. Covers every common pattern with explanation, DAX code, and discussion of edge cases: time intelligence, semi-additive measures, ABC analysis, basket analysis, statistical measures. Bookmark and use constantly.
   - **Time:** Read the Time Intelligence patterns first (~1 hour). Use as reference throughout.

2. **(Essential)** [Understanding CALCULATE — SQLBI](https://www.sqlbi.com/articles/the-calculate-function-in-dax/)
   - **Why:** CALCULATE is the most important function in DAX. This article explains how it modifies filter context step-by-step, with visual diagrams showing what the filter context looks like before and after CALCULATE.
   - **Time:** 30 min read

3. **(Recommended)** [ALL, ALLEXCEPT, ALLSELECTED — SQLBI](https://www.sqlbi.com/articles/using-allexcept-versus-all-and-values/)
   - **Why:** The difference between ALL, ALLEXCEPT, and ALLSELECTED is one of the most frequently misunderstood aspects of DAX. This article uses clear diagrams to show exactly what filter context each function creates.
   - **Time:** 20 min read

4. **(Recommended)** [Using Variables in DAX — SQLBI](https://www.sqlbi.com/articles/variables-in-dax/)
   - **Why:** Variables were introduced to DAX in 2015. They dramatically improve readability and sometimes performance. This article explains when VAR/RETURN helps (reused sub-expressions) and the subtle context evaluation rule.
   - **Time:** 15 min read

4. **(Advanced)** [Context Transition in DAX — SQLBI](https://www.sqlbi.com/articles/row-context-and-filter-context-in-dax/)
   - **Why:** Context transition (the conversion of row context to filter context when a measure is called inside an iterator) is the most subtle concept in DAX. Understanding it prevents a class of bugs that are very hard to diagnose otherwise.
   - **Time:** 40 min read

#### Videos

1. **(Essential)** [CALCULATE Deep Dive — SQLBI (Alberto Ferrari)](https://www.youtube.com/watch?v=erGpHq0kID0)
   - **Why:** The clearest explanation of filter context and CALCULATE available anywhere. Alberto Ferrari is one of the two foremost DAX experts in the world. This 90-minute video replaces weeks of self-study.
   - **Duration:** 90 min (watch in full — do not skip)

2. **(Essential)** [DAX Masterclass — SQLBI Course](https://www.sqlbi.com/courses/dax/)
   - **Why:** The industry-standard DAX course. Covers everything from fundamentals to advanced patterns. The free portion covers filter context and CALCULATE. The paid portion covers advanced patterns.
   - **Duration:** Free portion ~3 hours; full course ~10 hours

3. **(Recommended)** [Time Intelligence in DAX — Guy in a Cube](https://www.youtube.com/watch?v=cN9JxdmoBSc)
   - **Why:** Practical walkthrough of all time intelligence functions with live examples and common mistakes (fiscal year end parameter, comparison when prior year data is incomplete).
   - **Duration:** 35 min

#### Custom Resources

1. **(Essential)** **CALCULATE Filter Context Reference**

   ```
   CALCULATE(expression, filter1, filter2, ...)

   WHAT EACH FILTER TYPE DOES:
   ─────────────────────────────────────────────────────────────
   Boolean filter:
     CALCULATE([Revenue], dim_store[region] = "Northeast")
     → Replaces the region filter with "Northeast only"
     → Any existing slicer filter on region is OVERWRITTEN
     → Fast — use as default for simple column comparisons

   Table filter (FILTER function):
     CALCULATE([Revenue], FILTER(dim_store, dim_store[region] = "Northeast"))
     → Same result but slower — avoid for simple comparisons
     → Use ONLY when the condition references a measure or complex multi-column logic

   KEEPFILTERS modifier:
     CALCULATE([Revenue], KEEPFILTERS(dim_store[region] = "Northeast"))
     → Intersects with existing slicer filter (both must be true)
     → If slicer = "Southeast" and KEEPFILTERS = "Northeast" → BLANK (no overlap)

   ALL / REMOVEFILTERS modifier:
     CALCULATE([Revenue], ALL(dim_store))
     → Removes ALL filters from dim_store (including slicer filters)
     → Filters from other tables (dim_date, dim_product) are PRESERVED
     → Use for % of total calculations

   CALCULATE evaluation order:
   1. Evaluate the filter arguments (in filter context BEFORE CALCULATE modifies it)
   2. Modify the filter context with the filter arguments
   3. Evaluate the expression in the NEW filter context
   ```

2. **(Essential)** **Time Intelligence Prerequisites Checklist**

   ```
   BEFORE ANY TIME INTELLIGENCE MEASURE WILL WORK:

   □ A date table exists in the model
   □ The date table is marked as a Date Table
     (Power BI Desktop → Table Tools → Mark as date table)
   □ The date table has a continuous, gapless date range
     (no missing dates — even weekends and holidays must be present)
   □ The date range covers ALL dates in the fact table
     (if facts go back to 2021, date table must start at or before 2021-01-01)
   □ The date table has an active relationship to the fact table
     (via an integer DateKey column — not a date column)
   □ Auto date/time is disabled
     (File → Options → Global → Data Load → uncheck Auto date/time)

   COMMON TIME INTELLIGENCE FAILURES:
   ✗ DATESYTD returns BLANK for the current period
     → Date table doesn't extend to the refresh date
   ✗ SAMEPERIODLASTYEAR returns BLANK for all of Q1
     → The prior year period isn't in the date table
   ✗ YoY shows weird numbers in the first month of the year
     → Using DATEADD(-1, YEAR) instead of SAMEPERIODLASTYEAR
        (DATEADD shifts exact days; SAMEPERIODLASTYEAR handles year boundaries)
   ✗ DATESYTD shows wrong amounts for fiscal year
     → Missing the fiscal year end parameter ("6/30" for July 1 fiscal year start)
   ```

---

### C.3–C.4: Time Intelligence and Advanced Patterns

#### Articles

1. **(Essential)** [Semi-Additive Measures in DAX — SQLBI](https://www.sqlbi.com/articles/semi-additive-measures-in-dax/)
   - **Why:** Headcount, account balance, and inventory are the most common semi-additive measures in enterprise BI. This article explains the LASTDATE and LASTNONBLANK patterns with clear examples of when each is correct.
   - **Time:** 20 min read

2. **(Recommended)** [Pareto Analysis in DAX — DAX Patterns](https://www.daxpatterns.com/abc-classification/)
   - **Why:** The ABC (Pareto) classification is requested on nearly every retail, logistics, and product analytics project. The DAX Patterns implementation uses virtual tables — a technique that applies to many other advanced patterns.
   - **Time:** 25 min read

---

---

## MODULE D: Visualization, Security & Deployment

---

### D.1–D.2: Report Design and Interactivity

#### Articles

1. **(Essential)** [Power BI Report Design Best Practices — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/visuals/power-bi-report-visualizations)
   - **Why:** Official guidance on choosing visuals, layout, formatting, and accessibility. The accessibility section (tab order, alt-text, high-contrast support) is especially important for enterprise clients with compliance requirements.
   - **Time:** 30 min read

2. **(Essential)** [IBCS — International Business Communication Standards](https://www.ibcs.com/standards/)
   - **Why:** IBCS provides the most widely adopted framework for professional business report design. Key rules: use uniform scales, distinguish actual vs budget with solid vs outlined bars, use horizontal bar charts for many categories, avoid pie charts. Many large enterprises require IBCS-compliant reports.
   - **Time:** 20 min (read the "Notation" section)

3. **(Recommended)** [Drill-Through in Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/create-reports/desktop-drillthrough)
   - **Why:** Complete guide to drill-through configuration: setting up drill-through fields, keeping filters on drill-through, the Back button, and cross-report drill-through. Required reading before configuring drill-through in any report.
   - **Time:** 20 min read

#### Videos

1. **(Essential)** [Power BI Report Design from Scratch — Havens Consulting](https://www.youtube.com/watch?v=94BumC1CJ08)
   - **Why:** Professional report design tutorial showing layout, custom theme creation, consistent color application, and the difference between a "default" Power BI report and a polished enterprise dashboard.
   - **Duration:** 45 min

2. **(Recommended)** [Bookmarks and Buttons for Navigation — Guy in a Cube](https://www.youtube.com/watch?v=xCMqWEvSkAs)
   - **Why:** Practical walkthrough of using bookmarks and buttons to build navigation menus, toggle views, and reset slicers. These patterns are used in almost every enterprise Power BI report.
   - **Duration:** 25 min

---

### D.3: Row-Level Security

#### Articles

1. **(Essential)** [Row-Level Security (RLS) with Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/enterprise/service-admin-rls)
   - **Why:** The complete official RLS guide: static roles, dynamic roles using USERPRINCIPALNAME(), testing in Desktop with "View as", and assigning users/groups to roles in Power BI Service. All the information needed to implement RLS from scratch.
   - **Time:** 30 min read

2. **(Recommended)** [Dynamic Row-Level Security in Power BI — SQLBI](https://www.sqlbi.com/articles/dynamic-row-level-security-in-power-bi/)
   - **Why:** Covers the USERPRINCIPALNAME() pattern in depth, including the edge case where a user isn't in the mapping table, and the performance implications of using LOOKUPVALUE in an RLS filter.
   - **Time:** 20 min read

---

### D.4: Performance Optimization, Deployment & AI-Assisted Development

#### Articles

1. **(Essential)** [Power BI Performance Best Practices — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/guidance/power-bi-optimization)
   - **Why:** The official optimization guide covering model size, query reduction, visual optimization, and incremental refresh. The checklist at the end is a good pre-deployment review tool.
   - **Time:** 30 min read

2. **(Essential)** [Deployment Pipelines in Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/create-reports/deployment-pipelines-overview)
   - **Why:** The official guide to Power BI deployment pipelines: creating pipelines, assigning workspaces, deployment rules (connection string overrides per stage), access management. Required for enterprise deployments.
   - **Time:** 30 min read

3. **(Recommended)** [DAX Studio — Free Tool for DAX Analysis](https://daxstudio.org/)
   - **Why:** DAX Studio is the industry standard for DAX query analysis. Server Timings mode shows exactly how much time the formula engine vs storage engine takes. Essential for optimizing slow measures. Free and open source.
   - **Time:** 20 min to install and learn the basics

4. **(Advanced)** [Using AI Copilot in Power BI — Microsoft Docs](https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction)
   - **Why:** Power BI Copilot (available in Fabric Premium) can generate DAX measures, summarise reports, and create visuals from natural language. Understanding its capabilities and limitations prevents over-reliance on AI-generated DAX without review.
   - **Time:** 20 min read

#### Videos

1. **(Essential)** [DAX Studio Tutorial — SQLBI](https://www.youtube.com/watch?v=RXeIFJAJT9w)
   - **Why:** Walkthrough of DAX Studio with Server Timings: how to run a measure, interpret SE vs FE time, identify the bottleneck, and optimize. The tool most Power BI developers wish they had known about earlier.
   - **Duration:** 30 min

#### Custom Resources

1. **(Essential)** **AI-Assisted DAX Generation — Review Checklist**

   ```
   BEFORE ACCEPTING ANY AI-GENERATED DAX MEASURE:

   □ Is the denominator in DIVIDE handled correctly?
     - DIVIDE([numerator], [denominator], BLANK()) ← correct
     - [numerator] / [denominator] ← wrong (errors on 0, no BLANK)

   □ Are time intelligence functions using the correct fiscal year end?
     - DATESYTD(dim_date[Date]) ← assumes calendar year (Jan 1)
     - DATESYTD(dim_date[Date], "6/30") ← for July 1 fiscal year start

   □ Does the measure use VAR for any sub-expression referenced twice?
     - If the same CALCULATE(SUM...) appears twice → use VAR

   □ Is ALL or REMOVEFILTERS scoped correctly?
     - ALL(dim_store) removes ALL filters from dim_store
       (may accidentally remove filters you want to keep)
     - REMOVEFILTERS(dim_store[region_name]) removes only region filter
     - Which one is correct depends on the business definition

   □ For semi-additive measures (headcount, balance):
     - Is LASTDATE or LASTNONBLANK used instead of SUM?
     - AI frequently generates SUM(headcount) — always wrong

   □ Does YoY return BLANK (not 0) when no prior year data exists?
     - DIVIDE([cur] - [py], [py], BLANK()) ← correct
     - DIVIDE([cur] - [py], [py], 0) ← misleading (shows 0% growth where no data)

   □ Are filter arguments correct for the measure's business definition?
     - If the denominator should respect the year slicer but ignore region:
       need REMOVEFILTERS(dim_store[region_name]) NOT ALL(dim_store)

   □ Test the measure:
     - In a matrix with Year on rows and no slicer (check grand total)
     - In a matrix with a region slicer active (check regional filter applies)
     - For time intelligence: verify prior year row returns correct value
   ```

2. **(Essential)** **Prompt Templates for Power BI AI Assistance**

   ```
   TEMPLATE 1: Generate a DAX measure
   ─────────────────────────────────────────────────────────────────
   Generate a DAX measure for Power BI with this definition:

   Measure name: [Measure Name]
   Tables in the model: [list your fact and dimension tables]
   Relationships: [e.g., fact_sales → dim_date via OrderDateKey (active)]
   Business definition: [plain English description]
   Edge cases: [e.g., return BLANK if no prior year; exclude returns]
   Fiscal year: [e.g., starts July 1]

   Use VAR/RETURN structure. Use DIVIDE for all divisions.
   Use REMOVEFILTERS rather than ALL when scoping filters to specific columns.

   ─────────────────────────────────────────────────────────────────
   TEMPLATE 2: Generate Power Query M code
   ─────────────────────────────────────────────────────────────────
   Generate Power Query M code for:

   Source: [e.g., SQL Server table named sales.order_lines]
   Transformations needed:
     1. [e.g., filter to orders after 2021-01-01]
     2. [e.g., add a DateKey column as YYYYMMDD integer]
     3. [e.g., remove columns: created_by, modified_by, session_id]
     4. [e.g., set data types: order_id as Int64, amount as Currency]

   Note: Put all non-foldable transformations at the END of the chain.

   ─────────────────────────────────────────────────────────────────
   TEMPLATE 3: Generate RLS filter expression
   ─────────────────────────────────────────────────────────────────
   Generate a DAX RLS filter expression for Power BI.

   Security requirement: [e.g., each user should only see data for
                         their organisation, determined by their email]

   Security mapping table: security_mapping
   Columns: user_email (text), org_id (integer)

   The filter should be applied to the dim_organisation table
   on the org_id column.

   Use USERPRINCIPALNAME() for the email lookup.
   Handle the case where the user is not in the mapping table
   (they should see no data — return an impossible condition).
   ```
