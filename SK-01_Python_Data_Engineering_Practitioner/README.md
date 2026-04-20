# SK-01: Python Data Engineering Practitioner

> Build production-grade Python pipelines that ingest, clean, validate, and deliver data.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 40-50 hours |
| **Prerequisites** | Bridge Programme (or equivalent Python/pandas familiarity) |
| **Badge Earned** | Python Data Engineering Practitioner |
| **Difficulty** | Foundational |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Structure a Python project for maintainability and collaboration
- [ ] Ingest data from CSV, JSON, APIs, and fixed-width files
- [ ] Clean and standardize messy production data
- [ ] Deduplicate records across multiple sources
- [ ] Build reusable data quality validators
- [ ] Create visualizations for data profiling and reporting
- [ ] Deliver pipeline outputs in production-ready formats (Parquet, CSV)

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   1. RESOURCES          2. LAB              3. CAPSTONE         │
│   ─────────────         ─────────           ───────────         │
│   Learn the concepts    Apply them in a     Prove mastery       │
│   (20-25 hrs)           guided project      independently       │
│                         (6-8 hrs)           (12-15 hrs)         │
│                                                                 │
│   Module A: Foundations                                         │
│        ↓                                                        │
│   Module B: Pandas & NumPy                                      │
│        ↓                                                        │
│   Module C: Cleaning & Validation                               │
│        ↓                                                        │
│   Module D: EDA & Delivery                                      │
│        ↓                                                        │
│   Guided Lab: ShopStream Customer Pipeline                      │
│        ↓                                                        │
│   Capstone: GlobalTech HR Integration                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Recommended Order

### Week 1-2: Resources (Modules A-B)
Start with the **Essential** items in each module. Use **Recommended** to deepen understanding. Save **Advanced** for after you've completed the Lab.

### Week 3: Resources (Modules C-D) + Lab
Continue through the remaining modules, then complete the guided lab. The lab will reinforce everything you've learned.

### Week 4: Capstone
Work independently on the capstone project. This is where you prove your skills.

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Curated learning materials organized by module |
| [Labs/](Labs/) | Guided hands-on project (ShopStream) |
| [Capstone/](Project/) | Independent project for badge certification |

---

## Prerequisites Check

Before starting, confirm you can do the following:

```python
# Can you explain what this code does?
import pandas as pd

df = pd.read_csv("data.csv")
filtered = df[df["status"] == "active"]
result = filtered.groupby("region")["revenue"].sum()
result.to_csv("output.csv")
```

If this looks unfamiliar, complete the **Bridge Programme** first.

---

## Badge Criteria

To earn the **Python Data Engineering Practitioner** badge:

1. Complete all **Essential** resources in each module
2. Complete the guided Lab
3. Submit a passing Capstone project (see [Capstone/Rubric.md](Project/SK-01_Capstone_Project.md) for criteria)

---

## Next Skillset

After completing SK-01, proceed to:
→ **SK-02: SQL and Data Modeling Specialist**
