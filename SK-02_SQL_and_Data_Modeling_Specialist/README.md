# SK-02: SQL and Data Modeling Specialist

> Design dimensional models and write production-grade analytical SQL.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 25-35 hours |
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
│   RESOURCES              LAB                 CAPSTONE           │
│   (25-30 hrs)            (8-10 hrs)          (12-15 hrs)        │
│                                                                 │
│   Module A: SQL Foundations & Core Querying                     │
│        ↓                                                        │
│   Module B: Advanced SQL (Window Functions, CTEs)               │
│        ↓                                                        │
│   Module C: Dimensional Modeling & Star Schema                  │
│        ↓                                                        │
│   Module D: ETL Patterns & Performance                          │
│        ↓                                                        │
│   Guided Lab: E-Commerce Analytics Warehouse                    │
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

Can you write this query without help?

```sql
SELECT
    customer_id,
    SUM(amount) AS total_spent,
    COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 1000
ORDER BY total_spent DESC;
```

If not, review Module A resources thoroughly before proceeding.

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | Curated SQL and modeling materials |
| [Labs/](Labs/) | Guided e-commerce warehouse project |
| [Capstone/](Project/) | Healthcare claims data warehouse |

---

## Completion Criteria

To earn the **SQL and Data Modeling Specialist**:

1. Complete all **Essential** resources
2. Complete the guided Lab (e-commerce warehouse)
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
