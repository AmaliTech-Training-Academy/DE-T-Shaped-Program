# T-Shaped Data Engineering Programme

A production-focused, self-paced curriculum to transform career changers into job-ready data engineers.

---

## What This Is

12 skillsets that build real data engineering capabilities. Each skillset includes:

| Component | Purpose | Time |
|-----------|---------|------|
| **Resources** | Curated learning materials (no redundancy, production-focused) | 20-25 hrs |
| **Lab** | Guided hands-on project with a realistic business scenario | 6-8 hrs |
| **Capstone** | Independent project to prove mastery and earn a badge | 10-15 hrs |

---

## Start Here

### Step 1: Assess Your Starting Point

Can you do this without looking anything up?

```python
import pandas as pd
df = pd.read_csv("data.csv")
result = df[df["status"] == "active"].groupby("region")["revenue"].sum()
```

- **Yes** → Start at [SK-01: Python Data Engineering](SK-01_Python_Data_Engineering_Practitioner/)
- **No** → Start at [Bridge Programme](Bridge_Programme/)

### Step 2: Follow the Learning Path

See [LEARNING_PATH.md](LEARNING_PATH.md) for the full visual guide and dependencies.

---

## The 12 Skillsets

### Core Foundation (Everyone)
| # | Skillset | Focus |
|---|----------|-------|
| 01 | [Python Data Engineering Practitioner](SK-01_Python_Data_Engineering_Practitioner/) | Pipelines, cleaning, validation |
| 02 | [SQL and Data Modeling Specialist](SK-02_SQL_and_Data_Modeling_Specialist/) | Advanced SQL, dimensional modeling |
| 03 | [Batch ETL and Orchestration Engineer](SK-03_Batch_ETL_and_Orchestration_Engineer/) | Airflow, dbt, Spark |

### Platform Track: AWS
| # | Skillset | Focus |
|---|----------|-------|
| 04 | [AWS Cloud Data Engineer](SK-04_AWS_Cloud_Data_Engineer/) | S3, Glue, Athena, Redshift |
| 05 | [Real-Time Streaming Engineer](SK-05_Real_Time_Streaming_Engineer/) | Kafka, Kinesis, Lambda |

### Platform Track: Microsoft
| # | Skillset | Focus |
|---|----------|-------|
| 06 | [Power BI Analytics Practitioner](SK-06_Power_BI_Analytics_Practitioner/) | DAX, data modeling, reports |
| 07 | [Microsoft Fabric Data Engineer](SK-07_Microsoft_Fabric_Data_Engineer/) | Lakehouses, pipelines, notebooks |

### Advanced Layer
| # | Skillset | Focus |
|---|----------|-------|
| 08 | [Data Lakehouse Architect](SK-08_Data_Lakehouse_Architect/) | Delta Lake, Iceberg, medallion architecture |
| 09 | [Data Governance & Security Practitioner](SK-09_Data_Governance_and_Security_Practitioner/) | Access control, lineage, compliance |
| 10 | [DataOps and CI/CD Engineer](SK-10_DataOps_and_CI_CD_Engineer/) | Testing, deployment, monitoring |
| 11 | [NoSQL and DynamoDB Specialist](SK-11_NoSQL_and_DynamoDB_Specialist/) | Key-value, document stores, DynamoDB |
| 12 | [ML and Data Science for Data Engineers](SK-12_ML_and_Data_Science_for_Data_Engineers/) | Feature engineering, MLOps basics |

---

## How to Navigate Each Skillset

Every skillset folder contains:

```
SK-XX_Skillset_Name/
├── README.md           ← START HERE (overview, prerequisites, time estimate)
├── Resources/          ← Learn the concepts
│   ├── 00_Quick_Reference.md   ← One-page cheat sheet
│   └── Module_X_Topic.md       ← Detailed learning materials
├── Lab/                ← Apply knowledge in guided project
│   └── Guided_Lab.md
└── Capstone/           ← Prove mastery independently
    ├── Project.md
    └── Rubric.md       ← Evaluation criteria
```

---

## Time Commitment

| Path | Duration | Hours/Week |
|------|----------|------------|
| Full curriculum (AWS track) | ~9 months | 15 hrs/week |
| Full curriculum (Microsoft track) | ~9 months | 15 hrs/week |
| Core only (SK-01 to SK-03) | ~3 months | 15 hrs/week |

---

## Resource Philosophy

Every resource in this curriculum follows these principles:

1. **No redundancy** — One excellent resource per concept, not five okay ones
2. **Production-focused** — Learn what you'll actually use at work
3. **Tiered difficulty** — Essential → Recommended → Advanced
4. **Time-boxed** — Every resource has an estimated completion time
5. **Why before how** — Understand the business context before the syntax

---

## Badges

Complete a skillset's capstone to earn its badge. Badges are recorded and can be displayed on LinkedIn or your portfolio.

---

## Questions?

Each skillset README contains:
- Prerequisites check
- Learning outcomes
- Recommended order
- Badge criteria

Start with the README in any skillset folder.
