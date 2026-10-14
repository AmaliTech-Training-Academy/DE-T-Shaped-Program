# SK-01: Python Data Engineering Practitioner

> Build production-grade Python pipelines that ingest, clean, validate, and deliver data.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 25-35 hours |
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
│             guided project      independently       │
│                                            │
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

### Resources (Modules A-B)
Start with the **Essential** items in each module. Use **Recommended** to deepen understanding. Save **Advanced** for after you've completed the Lab.

### Resources (Modules C-D) + Lab
Continue through the remaining modules, then complete the guided lab. The lab will reinforce everything you've learned.

### Capstone
Work independently on the capstone project. This is where you prove your skills.

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Curated learning materials organized by module |
| [Labs/](Labs/) | Guided hands-on project (ShopStream) |
| [Granular Labs/](Granular%20Labs/) | Step-by-step labs that build up to the guided lab |
| [Capstone/](Project/) | Independent project for badge |

---

## Granular Labs

New to Python data pipelines? Work through these step-by-step labs **before or alongside** the guided lab. They use the same ShopStream scenario, data, file names and project folder (`C:\DataEngineering\sk01`). By Lab 06 your `pipeline.py` *is* the guided lab's pipeline, with tests on top.

| Lab | Topic | Guided lab coverage |
|---|---|---|
| [Lab 00](Granular%20Labs/Lab_00_Environment_Setup.md) | Environment setup: Python 3.12, venv, packages, VS Code | Prerequisites, project folder |
| [Lab 01](Granular%20Labs/Lab_01_Python_Foundations_for_Data_Engineering.md) | Python foundations on ShopStream values | The Python the pipeline relies on |
| [Lab 02](Granular%20Labs/Lab_02_Working_with_Files_and_Pandas.md) | Files and pandas: CSV, JSON, fixed-width, Excel | Parts 1–2 |
| [Lab 03](Granular%20Labs/Lab_03_Data_Cleaning_and_Validation.md) | Cleaning, deduplication, quality checks | Parts 3–5 |
| [Lab 04](Granular%20Labs/Lab_04_Building_a_Modular_ETL_Pipeline.md) | Orchestration, chart and delivery | Parts 7–8 |
| [Lab 05](Granular%20Labs/Lab_05_Logging_Error_Handling_and_Idempotency.md) | Logging, errors, CRM API, secrets, idempotency | Step 1.1, Step 2.2, Part 6, Reflection Q4–Q5 |
| [Lab 06](Granular%20Labs/Lab_06_Testing_and_Packaging_a_Production_Pipeline.md) | pytest tests and handover packaging | Tests for Parts 2–5 and 8 |

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

## Completion Criteria

To earn the **Python Data Engineering Practitioner**:

1. Complete all **Essential** resources in each module
2. Complete the guided Lab
3. Submit a passing Capstone project (see [Capstone/Rubric.md](Project/SK-01_Capstone_Project.md) for criteria)

---

## Next Skillset

After completing SK-01, proceed to:
→ **SK-02: SQL and Data Modeling Specialist**
