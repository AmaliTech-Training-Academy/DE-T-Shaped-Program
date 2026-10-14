# SK-06: Power BI Analytics Practitioner

> Build enterprise-grade reports and dashboards with proper data modeling.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 35-45 hours |
| **Prerequisites** | SK-03 Batch ETL and Orchestration Engineer |
| **Badge Earned** | Power BI Analytics Practitioner |
| **Difficulty** | Intermediate |
| **Platform** | Microsoft Track |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Design star schema models optimized for Power BI
- [ ] Write DAX measures for complex business calculations
- [ ] Build interactive reports with proper visual design
- [ ] Implement row-level security (RLS)
- [ ] Optimize model size using best practices
- [ ] Manage workspaces and deployment pipelines

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCES              LAB                 CAPSTONE           │
│   (20-25 hrs)            (6-8 hrs)           (10-12 hrs)        │
│                                                                 │
│   Module A: Data Modeling in Power BI                           │
│        ↓                                                        │
│   Module B: DAX Fundamentals                                    │
│        ↓                                                        │
│   Module C: Report Design & Visualization                       │
│        ↓                                                        │
│   Module D: Service, Security & Deployment                      │
│        ↓                                                        │
│   Granular Labs 00–06: step-by-step preparation                 │
│        ↓                                                        │
│   Guided Lab: NorthStar Executive Sales Dashboard               │
│        ↓                                                        │
│   Capstone: Executive KPI Dashboard Suite                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Focus Areas |
|--------|-------------|
| A | Star schema, relationships, Import vs DirectQuery, VertiPaq |
| B | CALCULATE, filter context, time intelligence, iterator functions |
| C | Visual best practices, conditional formatting, bookmarks, drillthrough |
| D | Workspaces, RLS, deployment pipelines, dataflows |

---

## DAX You'll Master

```dax
// Time Intelligence
Sales YTD = TOTALYTD([Total Sales], 'Date'[Date])
Sales PY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
YoY Growth = DIVIDE([Total Sales] - [Sales PY], [Sales PY])

// Filter Context
Sales Selected Region =
    CALCULATE([Total Sales], KEEPFILTERS(Region[Region]))
```

---

## Granular Labs

The [Granular Labs](Granular%20Labs/) are step-by-step, beginner-friendly labs that prepare you to complete the [End-to-End Guided Lab](Labs/SK-06_End_to_End_Guided_Lab.md) on your own. They use **exactly** the guided lab's environment (Power BI Desktop; PostgreSQL 16 in Docker, container `sk06-postgres`, database `northstar` on `localhost:5434`; project folder `C:\DataEngineering\sk06`), data (a seeded Python generator: 85 stores, 12,000 products, 310,024 order lines over FY2024–FY2026), queries and measures, in the same order. Every step has exact expected numbers, checked against SQL control totals (FY2026 net revenue $11,862,350.26).

| Lab | Title | Time | Guided lab coverage |
|---|---|---|---|
| 00 | [Environment Setup](Granular%20Labs/Lab_00_Environment_Setup.md) | 2–3 h | Part 0 |
| 01 | [Power BI & BI Concepts](Granular%20Labs/Lab_01_Power_BI_and_BI_Concepts.md) | 3–4 h | Scenario, Why this scenario, What you will build |
| 02 | [Power Query: Data Ingestion & Transformation](Granular%20Labs/Lab_02_Power_Query_Data_Ingestion_and_Transformation.md) | 5–6 h | Step 0.5, Part 2 (Steps 2.1–2.3) |
| 03 | [Data Modeling](Granular%20Labs/Lab_03_Data_Modeling.md) | 4–5 h | Part 1, Step 2.4 |
| 04 | [DAX Fundamentals](Granular%20Labs/Lab_04_DAX_Fundamentals.md) | 5–6 h | Step 3.1, % of company total (3.4) |
| 05 | [Advanced DAX & Time Intelligence](Granular%20Labs/Lab_05_Advanced_DAX_and_Time_Intelligence.md) | 5–6 h | Steps 3.2–3.4 |
| 06 | [Report Design, Dashboards, RLS, Performance & Publishing](Granular%20Labs/Lab_06_Report_Design_Dashboards_RLS_Performance_Publishing.md) | 6–7 h | Parts 4–7 |

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | SQLBI articles, DAX patterns, Microsoft docs |
| [Granular Labs/](Granular%20Labs/) | Step-by-step labs 00–06 that prepare you for the guided lab |
| [Labs/](Labs/) | Guided NorthStar Retail executive sales dashboard |
| [Capstone/](Project/) | Executive KPI dashboard suite |

---

## Badge Criteria

To earn the **Power BI Analytics Practitioner** badge:

1. Complete all **Essential** resources
2. Complete the guided Lab
3. Submit a passing Capstone project:
   - Star schema data model
   - 10+ DAX measures including time intelligence
   - Multi-page report with drillthrough
   - Row-level security implementation
   - Performance optimization documentation

---

## Next Skillset

After completing SK-06, proceed to:
→ **SK-07: Microsoft Fabric Data Engineer**
