# Module D: ETL Patterns & Performance

**Time:** 4-5 hours | **Focus:** Incremental loading, indexes, query optimization

---

## D.1 ETL Patterns

### Read

| Resource | Time | Why |
|----------|------|-----|
| [ETL Best Practices](https://towardsdatascience.com/etl-pipeline-best-practices-cc2cf5721c80) | 25 min | Idempotency and incremental patterns |

---

## D.2 Incremental Loading

### High-Water Mark Pattern

```sql
-- Get the last processed timestamp
SELECT MAX(processed_date) INTO v_watermark FROM fact_sales;

-- Load only new records
INSERT INTO fact_sales (...)
SELECT ...
FROM staging_sales s
JOIN dim_date d ON s.sale_date = d.date
WHERE s.processed_date > v_watermark;

-- Update watermark
UPDATE pipeline_metadata
SET last_run = CURRENT_TIMESTAMP
WHERE pipeline_name = 'sales_load';
```

### Upsert Pattern (MERGE)

```sql
-- PostgreSQL INSERT ON CONFLICT
INSERT INTO dim_product (product_id, name, category, updated_at)
VALUES ('P001', 'Widget', 'Hardware', CURRENT_TIMESTAMP)
ON CONFLICT (product_id) DO UPDATE SET
    name = EXCLUDED.name,
    category = EXCLUDED.category,
    updated_at = EXCLUDED.updated_at;

-- Standard SQL MERGE (if supported)
MERGE INTO dim_product t
USING staging_product s
ON t.product_id = s.product_id
WHEN MATCHED THEN
    UPDATE SET name = s.name, category = s.category
WHEN NOT MATCHED THEN
    INSERT (product_id, name, category) VALUES (s.product_id, s.name, s.category);
```

---

## D.3 Index Strategy

### When to Create Indexes

| Column Type | Index? | Why |
|-------------|--------|-----|
| Primary key | Auto | Unique constraint |
| Foreign key | Yes | JOIN performance |
| WHERE filter | Yes | Query performance |
| ORDER BY | Maybe | If frequently sorted |
| Low cardinality | No | Full scan often faster |

### Index Types

```sql
-- B-tree (default) - equality and range
CREATE INDEX idx_sales_date ON fact_sales(date_key);

-- Covering index - includes additional columns
CREATE INDEX idx_sales_covering ON fact_sales(date_key) INCLUDE (revenue);

-- Composite index - order matters!
CREATE INDEX idx_sales_product_date ON fact_sales(product_key, date_key);
-- This helps: WHERE product_key = 1 AND date_key > 20240101
-- Not helpful: WHERE date_key > 20240101 (first column not filtered)
```

---

## D.4 Query Optimization

### EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT p.category, SUM(f.revenue)
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
WHERE f.date_key BETWEEN 20240101 AND 20240131
GROUP BY p.category;
```

### Reading the Plan

```
GroupAggregate (cost=1234..5678 rows=100)
  -> Hash Join (cost=100..1000 rows=10000)
       Hash Cond: (f.product_key = p.product_key)
       -> Index Scan on fact_sales f (cost=0.42..500)
            Index Cond: (date_key >= 20240101 AND date_key <= 20240131)
       -> Hash (cost=50..50 rows=500)
            -> Seq Scan on dim_product p (cost=0..50)
```

**Key things to look for:**
- **Seq Scan on large tables** → Need an index
- **Hash Join** → Usually good for large tables
- **Nested Loop** → Good for small tables, bad for large
- **High cost** → Compare alternatives

---

## D.5 Materialized Views

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT
    d.year,
    d.month,
    p.category,
    SUM(f.revenue) AS total_revenue,
    COUNT(*) AS transaction_count
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.year, d.month, p.category;

-- Create index on materialized view
CREATE INDEX idx_mv_monthly_sales ON mv_monthly_sales(year, month);

-- Refresh (full rebuild)
REFRESH MATERIALIZED VIEW mv_monthly_sales;

-- Refresh concurrently (no lock, requires unique index)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales;
```

### When to Use

| Use Case | Materialized View? |
|----------|-------------------|
| Slow aggregate query run frequently | Yes |
| Real-time data needed | No |
| Dashboard with known refresh schedule | Yes |
| Ad-hoc exploration | No |

---

## Checkpoint

You are ready for the Lab when you can:

- [ ] Implement incremental loading with watermarks
- [ ] Create appropriate indexes for fact tables
- [ ] Read and interpret EXPLAIN ANALYZE output
- [ ] Create and refresh materialized views
