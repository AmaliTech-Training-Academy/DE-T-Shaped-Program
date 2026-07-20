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
│   Guided Lab: Sales Analytics Dashboard                         │
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

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | SQLBI articles, DAX patterns, Microsoft docs |
| [Labs/](Labs/) | Sales analytics dashboard project |
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
