# SK-04: AWS Cloud Data Engineer

> Design and build data platforms on AWS using S3, Glue, Athena, and Redshift.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 45-55 hours |
| **Prerequisites** | SK-03 Batch ETL and Orchestration Engineer |
| **Badge Earned** | AWS Cloud Data Engineer |
| **Difficulty** | Intermediate |
| **Platform** | AWS Track |

---

## What You'll Learn

By the end of this skillset, you will be able to:

- [ ] Design data lake architectures using S3 zones (raw/processed/curated)
- [ ] Configure IAM policies following least-privilege principles
- [ ] Build serverless ETL pipelines with AWS Glue
- [ ] Query S3 data using Athena with cost optimization
- [ ] Load and model data in Redshift
- [ ] Implement infrastructure as code with CloudFormation

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   RESOURCES              LAB                 CAPSTONE           │
│   (25-30 hrs)            (8-10 hrs)          (12-15 hrs)        │
│                                                                 │
│   Module A: AWS Architecture Foundations                        │
│        ↓                                                        │
│   Module B: S3 & Data Lake Design                               │
│        ↓                                                        │
│   Module C: AWS Glue & Serverless ETL                           │
│        ↓                                                        │
│   Module D: Athena & Redshift Analytics                         │
│        ↓                                                        │
│   Guided Lab: Serverless Analytics Platform                     │
│        ↓                                                        │
│   Capstone: Multi-Source Data Lake                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Topics

| Module | Focus Areas |
|--------|-------------|
| A | Regions/AZs, Well-Architected Framework, IAM, CloudFormation |
| B | S3 storage classes, partitioning, Glue Catalog, Lake Formation |
| C | Glue jobs, crawlers, job bookmarks, Glue Studio |
| D | Athena partitions, Redshift COPY, Spectrum, cost optimization |

---

## AWS Services Covered

```
S3 ──────────────→ Glue Catalog ──→ Athena (ad-hoc)
     │                   │
     └──→ Glue ETL ──────┴──────────→ Redshift (warehouse)
                                          │
Lambda ←── EventBridge ←── CloudWatch ←───┘
```

---

## Quick Links

| Section | Description |
|---------|-------------|
| [Resources/](Resources/) | AWS documentation and tutorials |
| [Labs/](Labs/) | Guided serverless analytics project |
| [Capstone/](Project/) | Multi-source data lake implementation |

---

## Badge Criteria

To earn the **AWS Cloud Data Engineer** badge:

1. Complete all **Essential** resources
2. Complete the guided Lab
3. Submit a passing Capstone project:
   - S3 bucket structure with proper zones
   - IAM roles with least-privilege policies
   - Glue ETL jobs with error handling
   - Athena queries with partition pruning
   - CloudFormation templates for infrastructure

---

## Next Skillset

After completing SK-04, proceed to:
→ **SK-05: Real-Time Streaming Engineer** (Kafka, Kinesis, Lambda)
