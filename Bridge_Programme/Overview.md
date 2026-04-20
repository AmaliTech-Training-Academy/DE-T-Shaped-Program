## My Assessment: 3 Real Problems to Solve

### Problem 1: The Entry Gap Is Bigger Than It Looks

Your learners have SQL and Python basics. Every skillset in this library is labelled "Intermediate" entry — but the actual assumed knowledge is much higher than that.

| Skillset        | What It Claims               | What It Actually Assumes                                     |
| --------------- | ---------------------------- | ------------------------------------------------------------ |
| SK-01 Python    | Entry: Beginner              | Comfortable with pandas, NumPy, writing functions, classes   |
| SK-02 SQL       | Entry: Beginner-Intermediate | Joins, window functions, CTEs, dimensional modelling concepts |
| SK-03 Batch ETL | Entry: Intermediate          | PySpark DataFrames, Airflow DAG structure, dbt basics        |
| SK-04 AWS       | Entry: Intermediate          | IAM, VPCs, S3, Lambda — AWS console familiarity              |

A learner with "SQL and Python basics" will hit a wall in the first week of SK-01 if we send them straight in. We need a bridge layer before the library starts.

------

### Problem 2: The Hours Math Is Very Tight

| Item                 | Hours                 |
| -------------------- | --------------------- |
| 12 skillsets total   | ~616 hours of content |
| 6 months of learning | ~26 weeks             |
| Required pace        | ~24 hours/week        |

That is achievable only if the learners are treating this as a near-full-time commitment. For a self-paced career changer who also has life obligations, 24 hrs/week is aggressive. At 15 hrs/week, they need 9 months. This is the honest number I think you should know before committing the cohort.

**My recommendation:** Either compress the programme to a "core 8" and make 4 skillsets electives, or extend to 8 months.

------

### Problem 3: The Sequencing Matters More Than the Content

The 12 skillsets cannot be delivered in any order. Some build directly on others. The wrong sequence creates confusion and dropout.

Here is the dependency map:

```
FOUNDATION (must come first)
    │
    ├── SK-01: Python Practitioner
    │       │
    │       └── SK-02: SQL & Data Modeling
    │               │
    │               └── SK-03: Batch ETL & Orchestration
    │                       │
    │          ┌────────────┴────────────┐
    │          │                         │
    │    PLATFORM TRACK A           PLATFORM TRACK B
    │    (AWS-focused)              (Microsoft-focused)
    │          │                         │
    │       SK-04: AWS              SK-06: Power BI
    │       SK-05: Streaming        SK-07: Fabric
    │          │                         │
    │          └────────────┬────────────┘
    │                       │
    │              ADVANCED LAYER
    │              (after platform track)
    │                       │
    │     ┌─────────────────┼─────────────────┐
    │     │                 │                 │
    │   SK-08          SK-09             SK-10
    │  Lakehouse      Governance         DataOps
    │  Architect      Security           CI/CD
    │                       │
    │                   SK-11
    │                  NoSQL/DynamoDB
    │                       │
    │                   SK-12
    │                  ML + Data Science
```

------

## What I Propose: A 3-Layer Architecture

### Layer 0: Bridge Programme (Weeks 1–3, before any skillset)

This does not exist yet and needs to be created. It covers:

- Python for data engineers (pandas, functions, classes, file I/O — not "Python basics")
- SQL beyond basics (window functions, CTEs, GROUP BY correctly)
- Cloud literacy (what is S3? what is a VM? what is an API?)
- Command line basics (terminal, git, environment variables)
- How the modern data stack fits together (conceptual overview)

This is the content that prevents the wall in week 1 of SK-01.

------

### Layer 1: Core Track (Weeks 4–16, Skillsets SK-01 to SK-03)

These three are mandatory for everyone regardless of specialisation. ~160 hours, 3 skillsets, ~4 weeks each.

------

### Layer 2: Platform Track (Weeks 17–22, one of two paths)

Learners choose AWS or Microsoft. They do not do both in 6 months.

**AWS Path:** SK-04 → SK-05 → SK-11 **Microsoft Path:** SK-06 → SK-07

------

### Layer 3: Advanced Layer (Weeks 23–26, shared)

SK-08 Lakehouse Architect + SK-09 Governance + SK-10 DataOps. SK-12 ML is the capstone.

