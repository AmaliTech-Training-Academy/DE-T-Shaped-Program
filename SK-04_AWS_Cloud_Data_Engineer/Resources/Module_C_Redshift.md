# Module C: Data Warehousing with Amazon Redshift

**Time:** 6-8 hours | **Focus:** Table design, distribution, loading, optimization

---

## C.1 Redshift Architecture

### Read

| Resource | Time | Why |
|----------|------|-----|
| [Redshift Table Design Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/c_designing-tables-best-practices.html) | 1 hr | Distribution styles, sort keys, compression |
| [Choosing Distribution Style](https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-best-dist-key.html) | 20 min | KEY vs ALL vs EVEN with examples |

### Watch

| Resource | Time | Why |
|----------|------|-----|
| [Redshift Deep Dive — re:Invent](https://www.youtube.com/watch?v=lj8oaSpCFTc) | 55 min | MPP, slices, COPY, WLM, Spectrum |

---

## C.2 Distribution Key Selection

### Decision Tree

```
TABLE SIZE?
├── Small (< 3M rows)
│   └── DISTSTYLE ALL
│       Replicate to every node. JOIN is always local.
│       Examples: dim_date, dim_country, reference tables
│
├── Large (> 3M rows)
│   └── DISTKEY or EVEN
│       │
│       ├── Frequent JOIN with another large table?
│       │   ├── YES: DISTKEY on JOIN column
│       │   │        Both tables must use same key
│       │   │
│       │   └── NO: EVEN distribution
│       │
│       └── IS THE KEY EVENLY DISTRIBUTED?
│           Check: SELECT distkey_col, COUNT(*) GROUP BY 1
│           If one value has >> 1/n_slices → DON'T use as DISTKEY
```

---

## C.3 Compression Encoding

### Reference

| Column Type | Best Encoding | Why |
|-------------|--------------|-----|
| Numbers (int, bigint, date) | AZ64 | AWS default, great for sorted |
| Low-cardinality strings | BYTEDICT | Dictionary encoding |
| High-cardinality strings | ZSTD | Best compression ratio |
| Sequential integers | DELTA | Stores differences |
| Repeated values | RUNLENGTH | Consecutive equal values |

### Let Redshift Choose

```sql
COPY my_table FROM 's3://...'
IAM_ROLE 'arn:aws:iam::...'
COMPUPDATE ON;

-- Or analyze existing table
ANALYZE COMPRESSION my_table;
```

---

## C.4 Loading & COPY Command

### Read

| Resource | Time | Why |
|----------|------|-----|
| [COPY Command Reference](https://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html) | 30 min | FORMAT, IAM_ROLE, MANIFEST |

### Template: COPY with Best Practices

```sql
COPY analytics.fact_orders
FROM 's3://bucket/orders/manifest.json'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftCopyRole'
FORMAT AS PARQUET
MANIFEST
COMPUPDATE ON
STATUPDATE ON;
```

### COPY Tips
- Use PARQUET format (columnar, compressed)
- Use MANIFEST for multiple files
- Enable COMPUPDATE for auto-compression
- Enable STATUPDATE for query planning

---

## C.5 Query Optimization

### Read

| Resource | Time | Why |
|----------|------|-----|
| [WLM Configuration Guide](https://docs.aws.amazon.com/redshift/latest/dg/cm-c-defining-query-queues.html) | 30 min | Query queues, memory, concurrency |

### Performance Tips

1. **ANALYZE** tables after large loads
2. **VACUUM** to reclaim space and re-sort
3. Use **EXPLAIN** to check execution plans
4. Check **svl_query_summary** for slow queries

---

## Checkpoint

Before moving to Module D, you should be able to:

- [ ] Choose the right distribution style for a table
- [ ] Select appropriate compression encodings
- [ ] Load data efficiently with COPY
- [ ] Configure WLM for workload isolation
