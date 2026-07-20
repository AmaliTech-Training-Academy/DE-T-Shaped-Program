# SK-07: Microsoft Fabric Data Engineer

> Build end-to-end analytics solutions on Microsoft Fabric's unified platform.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 45-55 hours |
| **Prerequisites** | SK-06 Power BI Analytics Practitioner |
| **Badge Earned** | Microsoft Fabric Data Engineer |
| **Difficulty** | Intermediate |
| **Platform** | Microsoft Track |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Navigate Fabric workspaces and understand OneLake architecture
- [ ] Build Lakehouses with medallion architecture (Bronze/Silver/Gold)
- [ ] Write Spark notebooks for data transformation
- [ ] Create data pipelines for orchestration
- [ ] Use shortcuts to virtualize external data
- [ ] Query data using the SQL Analytics Endpoint

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCES              LAB                 CAPSTONE           │
│   (25-30 hrs)            (8-10 hrs)          (12-15 hrs)        │
│                                                                 │
│   Module A: Fabric Platform & OneLake                           │
│        ↓                                                        │
│   Module B: Lakehouse & Delta Tables                            │
│        ↓                                                        │
│   Module C: Notebooks & Spark Processing                        │
│        ↓                                                        │
│   Module D: Pipelines & Data Warehouse                          │
│        ↓                                                        │
│   Guided Lab: E-Commerce Lakehouse                              │
│        ↓                                                        │
│   Capstone: Enterprise Data Platform                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Focus Areas |
|--------|-------------|
| A | Fabric experiences, OneLake, workspaces, shortcuts, capacity |
| B | Lakehouse creation, Delta Lake, MERGE, time travel, OPTIMIZE |
| C | PySpark in Fabric, notebook parameters, scheduling |
| D | Data Factory pipelines, Warehouse, Direct Lake, semantic models |

---

## Fabric Architecture

```
                        ┌─── Power BI Reports
                        │
Sources ──→ Lakehouse ──┼─── SQL Analytics Endpoint
   │         (Delta)    │
   │            │       └─── Warehouse (T-SQL)
   │            │
   └── Shortcuts ─────────→ OneLake (unified storage)
```

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Microsoft Fabric documentation |
| [Labs/](Labs/) | E-commerce lakehouse project |
| [Capstone/](Project/) | Enterprise data platform |

---

## Badge Criteria

To earn the **Microsoft Fabric Data Engineer** badge:

1. Complete all **Essential** resources
2. Complete the guided Lab
3. Submit a passing Capstone project:
   - Lakehouse with Bronze/Silver/Gold layers
   - Delta table maintenance (OPTIMIZE, VACUUM)
   - Spark notebooks with parameterization
   - Data pipeline with error handling
   - Power BI report using Direct Lake

---

## Next Skillsets

After completing SK-07, you can proceed to the Advanced Layer:
→ **SK-08: Data Lakehouse Architect**
→ **SK-09: Data Governance & Security Practitioner**
→ **SK-10: DataOps and CI/CD Engineer**
