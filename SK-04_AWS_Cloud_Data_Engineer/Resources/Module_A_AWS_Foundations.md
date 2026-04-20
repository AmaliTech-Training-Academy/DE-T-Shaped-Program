# Module A: AWS Architecture Foundations for Data Engineering

**Time:** 6-8 hours | **Focus:** Well-Architected Framework, IAM, VPC, cost management

---

## A.1 AWS Infrastructure & Well-Architected Framework

### Read

| Resource | Time | Why |
|----------|------|-----|
| [AWS Well-Architected — Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html) | 2 hrs | 5 pillars applied to data platforms |
| [AWS Global Infrastructure Guide](https://aws.amazon.com/about-aws/global-infrastructure/) | 20 min | Regions, AZs, edge locations |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [AWS re:Invent — Building Data Lakes on AWS](https://www.youtube.com/watch?v=Yp_4wkRbXJY) | 60 min | Full data lake architecture pattern |

### AWS Data Services Reference

```
SERVICE          PURPOSE                          PRICING MODEL
─────────────────────────────────────────────────────────────────
S3               Object storage (data lake)       $0.023/GB/month (Standard)
Kinesis Streams  Real-time ingestion buffer       $0.015/shard-hr + PUT units
Kinesis Firehose Managed delivery to S3/Redshift  $0.029/GB ingested
Glue             Serverless ETL + Data Catalog    $0.44/DPU-hour (ETL)
Athena           Serverless SQL on S3             $5/TB scanned
Redshift         Provisioned data warehouse       From $0.25/node-hour
EMR              Managed Spark/Hadoop cluster     EC2 + $0.18/vCPU-hr
Lambda           Event-driven compute             $0.0000166667/GB-second
```

---

## A.2 IAM — Identity, Access & Security

### Read

| Resource | Time | Why |
|----------|------|-----|
| [IAM Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) | 30 min | Evaluation order: Deny → SCP → boundary → policy |
| [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) | 20 min | Lock root, use roles, least privilege |

### IAM Policy Checklist

```
BEFORE WRITING ANY IAM POLICY:
□ What exact actions does this role need? (Not s3:*)
□ What specific resources? (Specific ARN, not *)
□ Are Condition keys needed? (s3:prefix, aws:RequestedRegion)

RED FLAGS — never accept without justification:
✗ "Action": "*"
✗ "Resource": "*"
✗ "Effect": "Allow" + "Action": "iam:*"
✗ Trust policy: "Principal": {"AWS": "*"}
```

---

## A.3 VPC & Networking

### Read

| Resource | Time | Why |
|----------|------|-----|
| [VPC Endpoints for S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html) | 20 min | Free, routes S3 traffic through AWS backbone |

### Key Point

**S3 Gateway Endpoints are mandatory** for data workloads:
- Free to use
- Routes S3 traffic through AWS backbone (not internet)
- Without it: $0.045/GB in NAT Gateway fees

---

## A.4 Infrastructure as Code

### Read

| Resource | Time | Why |
|----------|------|-----|
| [CloudFormation Getting Started](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html) | 45 min | Read + write basic templates |

### Template: S3 Bucket with Security

```yaml
Resources:
  DataLakeBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-datalake'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled
```

---

## Checkpoint

Before moving to Module B, you should be able to:

- [ ] Explain the AWS data services landscape
- [ ] Write least-privilege IAM policies
- [ ] Configure VPC endpoints for S3
- [ ] Create resources with CloudFormation
