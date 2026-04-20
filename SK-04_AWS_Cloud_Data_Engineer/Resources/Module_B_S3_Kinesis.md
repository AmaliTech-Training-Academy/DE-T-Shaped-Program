# Module B: Data Ingestion — S3 Data Lake & Kinesis

**Time:** 6-8 hours | **Focus:** S3 optimization, Kinesis streaming, partitioning

---

## B.1 Amazon S3 for Data Lakes

### Read

| Resource | Time | Why |
|----------|------|-----|
| [S3 Performance Guidelines](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html) | 20 min | Request rate limits, multipart upload |
| [S3 Lifecycle Transitions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html) | 25 min | Minimum durations, transition constraints |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [S3 Best Practices — re:Invent](https://www.youtube.com/watch?v=rHeTn9pHNKo) | 55 min | Performance, security, consistency |

### S3 Partitioning Framework

```
STEP 1: List query consumers and their filters
─────────────────────────────────────────────────────────────
Consumer A (ETL batch):    WHERE date = '2026-03-19'
Consumer B (analyst):      WHERE customer_id = 'cust-001'
Consumer C (dashboard):    WHERE date = CURRENT_DATE AND region = 'US'

STEP 2: Choose primary partition (serves highest-volume consumer)
─────────────────────────────────────────────────────────────
date/region → serves C well, A acceptably, B poorly

STEP 3: For poorly-served consumers
─────────────────────────────────────────────────────────────
If B queries > 100 times/day → secondary copy: customer_id/date/
If B queries < 10 times/day → tolerate the scan

ANTI-PATTERNS:
✗ Too many partition columns → millions of partitions
✗ High-cardinality first partition → millions of prefixes
✗ No partitioning → full scan every query
```

---

## B.2 Amazon Kinesis

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Kinesis Data Streams Best Practices](https://docs.aws.amazon.com/streams/latest/dev/kinesis-record-processor-scaling.html) | 30 min | Resharding, hot shard, checkpoint |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Kinesis Deep Dive — re:Invent](https://www.youtube.com/watch?v=MbEfiX4sMXc) | 45 min | Shard sizing, partition keys, KPL |

### Kinesis Shard Sizing

```
INPUTS:
Events per second (peak):    E events/sec
Average event size:          S bytes/event

CALCULATIONS:
Write capacity needed:       E ÷ 1,000 shards (record rate limit)
                            (E × S) ÷ 1MB shards (throughput limit)
Required shards = MAX(both calculations)

HEADROOM RULE: Add 25% above peak

COST COMPARISON (2 shards example):
PROVISIONED: ~$36/month (predictable workload)
ON_DEMAND:   ~$171/month (variable workload)
→ PROVISIONED wins for steady, predictable loads
```

---

## B.3 Storage Classes & Lifecycle

### Reference

| Storage Class | Use Case | Minimum Duration |
|---------------|----------|------------------|
| Standard | Frequently accessed | None |
| Standard-IA | Infrequent (30+ days) | 30 days |
| Glacier IR | Archive, fast retrieval | 90 days |
| Glacier Deep | Long-term archive | 180 days |

### Lifecycle Policy Pattern

```json
{
  "Rules": [
    {
      "ID": "ArchiveOldData",
      "Status": "Enabled",
      "Filter": { "Prefix": "raw/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER_IR" }
      ]
    }
  ]
}
```

---

## Checkpoint

Before moving to Module C, you should be able to:

- [ ] Design S3 partitioning for query patterns
- [ ] Calculate Kinesis shard requirements
- [ ] Configure S3 lifecycle policies
- [ ] Choose between Provisioned and On-Demand Kinesis
