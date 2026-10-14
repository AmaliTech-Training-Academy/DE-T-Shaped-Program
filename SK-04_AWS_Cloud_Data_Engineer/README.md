# SK-04: AWS Cloud Data Engineer

> Design and build data platforms on AWS using S3, Glue, Athena, and Redshift.

---

## At a Glance

| | |
|---|---|
| **Time to Complete** | 40-50 hours |
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

## Granular Labs

The granular labs take a beginner, one small step at a time, to the point where they can do the end-to-end guided lab on their own. They use the guided lab's scenario (CloudMetrics), its scripts (`scripts/cm_config.py`, `rs.py`, `10_cost_controls.py`, `99_teardown.py`, ...) and its order. These labs create real AWS resources: finish Lab 07's teardown when you are done.

| Lab | Title | Guided lab coverage |
|-----|-------|---------------------|
| 00 | [Environment Setup](Granular%20Labs/Lab_00_Environment_Setup.md) | Prerequisites, Part 0 |
| 01 | [AWS Foundations, IAM and boto3](Granular%20Labs/Lab_01_AWS_Foundations_IAM_and_Boto3.md) | Part 2, Step 2.1 (roles, trust and inline policies) |
| 02 | [S3 Data Lake with KMS Encryption and Lifecycle](Granular%20Labs/Lab_02_S3_Data_Lake_KMS_and_Lifecycle.md) | Part 1, Steps 1.1–1.2 |
| 03 | [Kinesis Data Streams and a Resilient Producer](Granular%20Labs/Lab_03_Kinesis_Data_Streams_and_Producer.md) | Part 3, Steps 3.1 and 3.4 |
| 04 | [Glue Data Catalog, CloudFormation and Firehose](Granular%20Labs/Lab_04_Glue_Catalog_CloudFormation_and_Firehose.md) | Part 2, Step 2.1; Part 3, Steps 3.2–3.4 |
| 05 | [Redshift Cluster, Networking and Table Design](Granular%20Labs/Lab_05_Redshift_Cluster_and_Table_Design.md) | Part 4, Steps 4.1–4.2 |
| 06 | [Loading Redshift: COPY, Manifests and Spectrum](Granular%20Labs/Lab_06_Redshift_COPY_Manifests_and_Spectrum.md) | Part 4, Steps 4.3–4.4 |
| 07 | [Cost Controls, Monitoring and Cost Analysis](Granular%20Labs/Lab_07_Cost_Controls_Monitoring_and_Cost_Analysis.md) | Parts 5–7 (cost controls, cost analysis, teardown) |

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
| [Granular Labs/](Granular%20Labs/) | Step-by-step labs 00–07 leading up to the guided lab |
| [Labs/](Labs/) | Guided lab: CloudMetrics AWS data platform |
| [Capstone/](Project/) | Multi-source data lake implementation |

---

## Completion Criteria

To earn the **AWS Cloud Data Engineer**:

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
