# SK-02: SQL and Data Modeling Specialist

> Design dimensional models and write production-grade analytical SQL.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 65-80 hours (resources 25-30, granular labs 20-24, guided lab 8-10, capstone 12-15) |
| **Prerequisites** | SK-01 Python Data Engineering Practitioner |
| **Badge Earned** | SQL and Data Modeling Specialist |
| **Difficulty** | Foundational |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Write complex analytical queries using window functions and CTEs
- [ ] Design star schema dimensional models from business requirements
- [ ] Implement Slowly Changing Dimensions (Type 1 and Type 2)
- [ ] Build and maintain fact tables at the correct grain
- [ ] Optimize queries using indexes and execution plan analysis
- [ ] Create materialized views for performance-critical reports

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCES       GRANULAR LABS     GUIDED LAB     CAPSTONE     │
│   (25-30 hrs)     (20-24 hrs)       (8-10 hrs)     (12-15 hrs)  │
│                                                                 │
│   Module A: SQL Foundations & Core Querying                     │
│        ↓                                                        │
│   Module B: Advanced SQL (Window Functions, CTEs)               │
│        ↓                                                        │
│   Module C: Dimensional Modeling & Star Schema                  │
│        ↓                                                        │
│   Module D: ETL Patterns & Performance                          │
│        ↓                                                        │
│   Granular Labs 00–08: step-by-step preparation                 │
│        ↓                                                        │
│   Guided Lab: FreshMart Retail Analytics Warehouse              │
│        ↓                                                        │
│   Capstone: MedInsure Healthcare Claims Warehouse               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Focus Areas |
|--------|-------------|
| A | SELECT, JOINs, filtering, NULL handling, execution order |
| B | Window functions (ROW_NUMBER, RANK, LAG/LEAD), CTEs, subqueries |
| C | Kimball methodology, star schema, fact/dimension design, grain |
| D | SCD Type 1 & 2, incremental loading, indexes, EXPLAIN ANALYZE |

---

## Prerequisites Check

Can you read this query and explain what it returns? (It runs on the FreshMart source system you build in Granular Lab 01.)

```sql
SELECT
    o.customer_id,
    SUM(oi.quantity * oi.unit_price - oi.discount_amount) AS total_spent,
    COUNT(DISTINCT o.order_id)                            AS order_count
FROM source_oltp.orders o
JOIN source_oltp.order_items oi ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY o.customer_id
HAVING SUM(oi.quantity * oi.unit_price - oi.discount_amount) > 1000
ORDER BY total_spent DESC;
```

If not, start with Module A resources and Granular Labs 00–02 before going further.

---

## Granular Labs

The [Granular Labs](Granular%20Labs/) are step-by-step, beginner-friendly labs that prepare you to complete the [End-to-End Guided Lab](Labs/SK-02_End_to_End_Guided_Lab.md) on your own. They use **exactly** the guided lab's environment (PostgreSQL 16 in Docker, container `sk02-postgres`, database `freshmart`, project folder `C:\DataEngineering\sk02`), schema (`source_oltp` source tables, star schema in `public`) and scripts, in the same order. Every step has an exact expected output from the deterministic sample data (100,000 orders / 300,235 order lines).

| Lab | Title | Time | Guided lab coverage |
|---|---|---|---|
| 00 | [Environment Setup](Granular%20Labs/Lab_00_Environment_Setup.md) | 60–90 min | Prerequisites |
| 01 | [Building the FreshMart Source System](Granular%20Labs/Lab_01_Building_the_FreshMart_Source_System.md) | 120–150 min | Part 0, Step 1.1 |
| 02 | [SQL Querying Foundations](Granular%20Labs/Lab_02_SQL_Querying_Foundations.md) | 150–180 min | Step 1.2 (the report) |
| 03 | [Query Plans and Indexes](Granular%20Labs/Lab_03_Query_Plans_and_Indexes.md) | 120–150 min | Step 1.2 (EXPLAIN), basics of 6.1 |
| 04 | [Dimensional Modeling and the Warehouse DDL](Granular%20Labs/Lab_04_Dimensional_Modeling_and_Warehouse_DDL.md) | 120–150 min | Part 2 |
| 05 | [Loading the Dimensions](Granular%20Labs/Lab_05_Loading_the_Dimensions.md) | 150–180 min | Part 3 |
| 06 | [Loading the Fact Table](Granular%20Labs/Lab_06_Loading_the_Fact_Table.md) | 150–180 min | Part 4 |
| 07 | [Analytical SQL and Window Functions](Granular%20Labs/Lab_07_Analytical_SQL_and_Window_Functions.md) | 180–210 min | Part 5 |
| 08 | [Performance: Indexes, MVs and Partitioning](Granular%20Labs/Lab_08_Performance_Indexes_MVs_Partitioning.md) | 120–150 min | Part 6 |

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Curated SQL and modeling materials |
| [Granular Labs/](Granular%20Labs/) | Step-by-step labs 00–08 that prepare you for the guided lab |
| [Labs/](Labs/) | Guided FreshMart retail warehouse project |
| [Capstone/](Project/) | Healthcare claims data warehouse |

---

## Completion Criteria

To earn the **SQL and Data Modeling Specialist**:

1. Complete all **Essential** resources
2. Complete the Granular Labs (recommended) and the guided Lab (FreshMart retail warehouse)
3. Submit a passing Capstone project (MedInsure)
   - Dimensional model design document
   - Physical DDL for all tables
   - ETL SQL for SCD Type 2 dimensions
   - 8 analytical queries with window functions
   - Performance optimization report

---

## Next Skillset

After completing SK-02, proceed to:
→ **SK-03: Batch ETL and Orchestration Engineer**
