# Lab 08 — Performance: Indexes, Materialized Views and Partitioning

> **Goal:** Make warehouse queries fast for dashboards: prove a covering index with before/after plans, pre-aggregate with a refreshable materialized view, and range-partition the fact table so queries prune whole years.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 07
> **Guided lab coverage:** PART 6 (Steps 6.1–6.3) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Run a disciplined tuning loop: **baseline plan → one change → new plan → same result**.
- Explain composite and **covering** (`INCLUDE`) indexes and when PostgreSQL can do an **Index Only Scan** (and why `VACUUM` matters for it).
- Create a **materialized view**, give it a unique index, and refresh it with `REFRESH MATERIALIZED VIEW CONCURRENTLY`.
- Explain staleness and when to refresh (after the nightly ETL).
- Create a **range-partitioned** table, explain why its primary key must include the partition key, and read **partition pruning** in a plan.

**Key terms**

| Term | Meaning |
|---|---|
| **Covering index** | An index that contains every column a query needs (`INCLUDE (…)` adds non-key columns to the index leaves), so the table itself is never read. |
| **Visibility map** | PostgreSQL's record of which table pages contain only rows visible to everyone. Index Only Scans need it up to date — `VACUUM` maintains it. |
| **Materialized view (MV)** | A saved query whose **result is stored** like a table: instant reads, but stale until refreshed. |
| **Partitioning** | Splitting one logical table into child tables by a key range (here: one partition per year). Queries that filter on the key skip (**prune**) irrelevant partitions. |

## Steps

Work in an interactive session with `\timing on`. At the end, save the statements of Steps 1–5 as `C:\DataEngineering\sk02\sql\11_performance.sql` (that file is the guided lab's Part 6).

### Step 1 — Baseline plan (guided Step 6.1, part 1)

The dashboard question: *net revenue by region for Q4 2025*. It filters on the fact's own `date_key` — a quarter is ~11% of the fact table. The fact already has the FK indexes from Lab 04 (`idx_fact_date`, `idx_fact_store_date`, …) but no covering index.

```sql
DROP INDEX IF EXISTS idx_fact_date_store_covering;   -- make sure we measure the true baseline

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
WHERE fs.date_key BETWEEN 20251001 AND 20251231      -- Q4 2025
GROUP BY ds.region_name;
```

**Expected output** (shape; timings vary): a `HashAggregate` over a `Hash Join` with `dim_store`, where `fact_sales` is read either by `Seq Scan on fact_sales` (`Filter: ((date_key >= 20251001) AND (date_key <= 20251231))`, about 251,000 `Rows Removed by Filter`) or by a `Bitmap Heap Scan` using `idx_fact_date` — either way **the table heap is read**. Note the `Execution Time` and the `Buffers: shared hit/read` numbers.

Run the query itself (without `EXPLAIN`) to record the answer:

```sql
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
WHERE fs.date_key BETWEEN 20251001 AND 20251231
GROUP BY ds.region_name
ORDER BY ds.region_name;
```

**Expected output:**

```
 region_name |  revenue
-------------+-----------
 Midwest     | 470619.53
 Northeast   | 471159.56
 Southeast   | 482082.61
 West        | 480016.26
(4 rows)
```

✅ **Checkpoint:** you have a baseline plan and timing written down.

### Step 2 — Covering index, statistics and VACUUM (guided Step 6.1, part 2)

```sql
ANALYZE fact_sales;
ANALYZE dim_date;
ANALYZE dim_store;

CREATE INDEX idx_fact_date_store_covering
    ON fact_sales(date_key, store_key)
    INCLUDE (net_revenue, quantity, gross_margin);

VACUUM (ANALYZE) fact_sales;

EXPLAIN (ANALYZE, BUFFERS)
SELECT ds.region_name, SUM(fs.net_revenue) AS revenue
FROM fact_sales fs
JOIN dim_store ds ON fs.store_key = ds.store_key
WHERE fs.date_key BETWEEN 20251001 AND 20251231
GROUP BY ds.region_name;
```

**Expected output:** `ANALYZE` ×3, `CREATE INDEX`, `VACUUM`, then a plan containing

```
->  Index Only Scan using idx_fact_date_store_covering on fact_sales fs  (… rows=31043 loops=1)
      Index Cond: ((date_key >= 20251001) AND (date_key <= 20251231))
      Heap Fetches: 0
```

and a lower `Execution Time` and buffer count than the baseline. Rerun the plain query from Step 1 — **same four revenue numbers**. A tuning change that alters results is a bug, not an optimisation.

Why it works:
- **Composite key** `(date_key, store_key)`: the range filter on `date_key` (the leading column) finds Q4 quickly; `store_key` is right there for the join.
- **`INCLUDE (net_revenue, …)`**: the summed column lives in the index leaves, so PostgreSQL never visits the table (`Index Only Scan`).
- **`VACUUM`** marked the pages all-visible; before it, the same plan would show thousands of `Heap Fetches` (each one a table visit).

✅ **Checkpoint:** your plan shows `Index Only Scan … idx_fact_date_store_covering` with `Heap Fetches: 0` (if the planner still prefers a Seq Scan on your machine, the index is not wrong — check the costs; the planner chose what it estimated cheaper).

### Step 3 — A materialized view for the dashboard (guided Step 6.2)

Dashboards ask the same grouped questions all day. Pre-compute them once per night:

```sql
-- 6.2 Materialized view
DROP MATERIALIZED VIEW IF EXISTS mv_daily_store_category_sales;
CREATE MATERIALIZED VIEW mv_daily_store_category_sales AS
SELECT
    fs.date_key,
    dd.year_month,
    dd.year_number,
    dd.month_number,
    dd.fiscal_year,
    dd.fiscal_quarter,
    ds.store_key,
    ds.store_name,
    ds.region_name,
    ds.district_name,
    dp.category_name,
    COUNT(*)                    AS line_item_count,
    COUNT(DISTINCT fs.order_id) AS order_count,
    SUM(fs.quantity)            AS total_units,
    SUM(fs.gross_revenue)       AS gross_revenue,
    SUM(fs.net_revenue)         AS net_revenue,
    SUM(fs.gross_margin)        AS gross_margin,
    SUM(fs.discount_amount)     AS total_discounts
FROM      fact_sales   fs
JOIN dim_date    dd ON fs.date_key    = dd.date_key
JOIN dim_store   ds ON fs.store_key   = ds.store_key
JOIN dim_product dp ON fs.product_key = dp.product_key
GROUP BY
    fs.date_key, dd.year_month, dd.year_number, dd.month_number,
    dd.fiscal_year, dd.fiscal_quarter,
    ds.store_key, ds.store_name, ds.region_name, ds.district_name,
    dp.category_name;

CREATE UNIQUE INDEX idx_mv_daily_sales
    ON mv_daily_store_category_sales(date_key, store_key, category_name);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_store_category_sales;

SELECT COUNT(*) AS mv_rows FROM mv_daily_store_category_sales;

EXPLAIN SELECT region_name, SUM(net_revenue)
FROM mv_daily_store_category_sales
WHERE year_number = 2025
GROUP BY region_name;
```

**Expected output:**

```
DROP MATERIALIZED VIEW
SELECT 169128
CREATE INDEX
REFRESH MATERIALIZED VIEW
 mv_rows
---------
  169128
(1 row)

                        QUERY PLAN
-----------------------------------------------------------
 HashAggregate  (…)
   Group Key: region_name
   ->  Seq Scan on mv_daily_store_category_sales  (…)
         Filter: (year_number = 2025)
```

(PostgreSQL may parallelise either plan — `Gather` / `Partial HashAggregate` / `Parallel Seq Scan` — on larger machines; the scanned relation is what matters. The first `DROP MATERIALIZED VIEW` prints a `NOTICE: … does not exist, skipping` the very first time.) 169,128 rows = one per (day, store, category) combination that had sales — far fewer than the 282,261 fact rows, already joined and summed.

- `REFRESH MATERIALIZED VIEW` recomputes the stored result. Plain refresh **locks out readers** while it runs; `CONCURRENTLY` lets dashboards keep reading, but needs a **unique index** on the MV — that is why `idx_mv_daily_sales` exists.
- An MV is **stale** between refreshes: schedule the refresh right after the nightly ETL (scripts 05–08), and tell users "data as of last night".

Prove the MV gives the same answer as the fact table:

```sql
SELECT region_name, SUM(net_revenue) AS revenue
FROM mv_daily_store_category_sales
WHERE year_number = 2025
GROUP BY region_name
ORDER BY region_name;
```

**Expected output:**

```
 region_name |  revenue
-------------+------------
 Midwest     | 1903697.76
 Northeast   | 1879383.57
 Southeast   | 1901636.85
 West        | 1914881.08
(4 rows)
```

(Same totals as Lab 07 Step 1.)

✅ **Checkpoint:** 169128 MV rows; region totals match Lab 07.

### Step 4 — Why `LIKE … INCLUDING ALL` fails for a partitioned table

The obvious way to clone the fact table as a partitioned table doesn't work — see why:

```sql
CREATE TABLE fact_sales_partitioned (
    LIKE fact_sales INCLUDING ALL
) PARTITION BY RANGE (date_key);
```

**Expected output:**

```
ERROR:  unique constraint on partitioned table must include all partitioning columns
DETAIL:  … constraint on table "fact_sales_partitioned" lacks column "date_key" which is part of the partition key.
```

Each partition enforces uniqueness on its own, so PostgreSQL can only guarantee a unique/primary key across partitions if the key *contains* the partition column. `INCLUDING ALL` copies `PRIMARY KEY (sales_key)` (and the `UNIQUE (order_id, order_line_number)`), neither of which contains `date_key`.

✅ **Checkpoint:** you can explain the error in one sentence.

### Step 5 — Range partitioning and pruning (guided Step 6.3)

Copy only columns, defaults and checks, then add a primary key that includes the partition key:

```sql
-- 6.3 Partitioning
DROP TABLE IF EXISTS fact_sales_partitioned CASCADE;
CREATE TABLE fact_sales_partitioned (
    LIKE fact_sales INCLUDING DEFAULTS INCLUDING CONSTRAINTS
) PARTITION BY RANGE (date_key);

ALTER TABLE fact_sales_partitioned
    ADD CONSTRAINT pk_fact_sales_partitioned PRIMARY KEY (sales_key, date_key);

CREATE TABLE fact_sales_2023 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20230101) TO (20240101);
CREATE TABLE fact_sales_2024 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20240101) TO (20250101);
CREATE TABLE fact_sales_2025 PARTITION OF fact_sales_partitioned
    FOR VALUES FROM (20250101) TO (20260101);
CREATE TABLE fact_sales_default PARTITION OF fact_sales_partitioned
    DEFAULT;

INSERT INTO fact_sales_partitioned SELECT * FROM fact_sales;
ANALYZE fact_sales_partitioned;

SELECT tableoid::regclass AS partition, COUNT(*)
FROM fact_sales_partitioned GROUP BY 1 ORDER BY 1;

EXPLAIN SELECT COUNT(*) FROM fact_sales_partitioned
WHERE date_key BETWEEN 20250101 AND 20251231;
```

**Expected output:**

```
DROP TABLE
CREATE TABLE
ALTER TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
INSERT 0 282261
ANALYZE
     partition      | count
--------------------+--------
 fact_sales_2023    |  59649
 fact_sales_2024    |  98430
 fact_sales_2025    | 124180
 fact_sales_default |      2
(4 rows)

                              QUERY PLAN
----------------------------------------------------------------------
 Aggregate  (…)
   ->  Seq Scan on fact_sales_2025 fact_sales_partitioned  (…)
         Filter: ((date_key >= 20250101) AND (date_key <= 20251231))
```

- `PARTITION BY RANGE (date_key)` + `FOR VALUES FROM (20250101) TO (20260101)` — lower bound inclusive, upper bound exclusive (a half-open range again).
- `INSERT` into the parent routes each row to the right child automatically; `tableoid::regclass` shows which child a row lives in.
- **Pruning:** the plan touches only `fact_sales_2025` (possibly as a `Parallel Seq Scan` under `Gather` / `Finalize Aggregate`); the other three partitions don't appear at all (with a single survivor PostgreSQL even drops the `Append` node). At production scale this turns "scan 15M rows" into "scan one year".
- The 2 rows in `fact_sales_default` are Lab 06's day-2 order (dated today). In production you create next year's partition **before** the year starts, so the default partition stays empty.

✅ **Checkpoint:** four partitions with the counts above, and the plan names only `fact_sales_2025`.

### Step 6 — Save the guided lab's Part 6 script

Combine Steps 1–3 and 5 into `C:\DataEngineering\sk02\sql\11_performance.sql` (baseline → ANALYZE → covering index → VACUUM → re-EXPLAIN → MV → partitioning) and run it once end to end:

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/11_performance.sql
```

**Expected output:** the same outputs as Steps 1, 2, 3 and 5 in sequence (no errors — the script drops and recreates its own objects, so it is rerunnable).

✅ **Checkpoint:** the script runs twice in a row without errors. You have now practised every step of the guided lab — go and do it end to end on your own.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Plan shows `Index Only Scan` but many `Heap Fetches` | Visibility map not updated after the load | `VACUUM (ANALYZE) fact_sales;` |
| Planner ignores the new index | Statistics stale, or the range is too large to be worth it | `ANALYZE fact_sales;`; compare with a narrower range (e.g. one month). |
| `cannot refresh materialized view "…" concurrently` | No unique index on the MV (or MV never populated) | Create `idx_mv_daily_sales`; run a plain `REFRESH` once. |
| `relation "idx_fact_date_store_covering" already exists` | Step rerun | The script starts with `DROP INDEX IF EXISTS`. |
| `no partition of relation "fact_sales_partitioned" found for row` | No default partition and a date outside the ranges | Keep the `DEFAULT` partition, or add the missing year's partition. |
| MV numbers differ from the fact table | MV not refreshed after the latest ETL | `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_store_category_sales;` |

## Key takeaways

- Tune with evidence: baseline plan, one change, new plan, identical results.
- Covering index = composite key for the filter/join + `INCLUDE` for the measures → Index Only Scan (after `VACUUM`).
- Materialized views trade freshness for speed; a unique index enables `REFRESH … CONCURRENTLY` — refresh after every ETL run.
- Partitioned tables need the partition key in every unique key; filters on the key prune whole partitions.

## Next

You are ready for the [SK-02 End-to-End Guided Lab](../Labs/SK-02_End_to_End_Guided_Lab.md): build the whole FreshMart warehouse from an empty database, on your own, in one sitting.
