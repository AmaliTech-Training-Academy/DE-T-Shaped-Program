# Lab 03 — Query Plans and Indexes

> **Goal:** Read PostgreSQL execution plans with `EXPLAIN ANALYZE`, decide when an index helps (and when it can't), and diagnose the guided lab's slow report.
> **Time:** 120–150 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 02
> **Guided lab coverage:** PART 1 Step 1.2 (EXPLAIN diagnosis) and the foundations of PART 6 Step 6.1 of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Explain what the **query planner** does and why it needs **statistics** (`ANALYZE`).
- Run `EXPLAIN`, `EXPLAIN ANALYZE` and `EXPLAIN (ANALYZE, BUFFERS)` and read the plan tree (innermost node first).
- Recognise `Seq Scan`, `Index Scan`, `Bitmap Heap Scan`, `Index Only Scan`, `Hash Join`, `Nested Loop`, `Sort`, `HashAggregate`/`GroupAggregate`.
- Create single-column and composite B-tree indexes and apply the column-order rule.
- Explain why low-selectivity filters and function-wrapped columns don't use indexes, and why indexes slow down writes.

**Key terms**

| Term | Meaning |
|---|---|
| **Planner** | The part of PostgreSQL that picks *how* to run your declarative SQL (scan order, join algorithm, index use) by estimating the cost of alternatives. |
| **Statistics** | Row counts and value distributions per column, gathered by `ANALYZE` (and automatically by autovacuum). Bad statistics → bad plans. |
| **Index** | A separate sorted structure (a **B-tree** by default) mapping column values → row locations, like the index of a book. |
| **Selectivity** | The fraction of rows a filter keeps. An index pays off for *selective* filters (few rows), not for filters that keep most of the table. |

## Steps

Use an interactive session and turn on timing first:

```sql
\timing on
```

**Expected output:** `Timing is on.`

### Step 1 — Your first plan: a Seq Scan

```sql
EXPLAIN ANALYZE
SELECT * FROM source_oltp.orders WHERE customer_id = 42;
```

**Expected output** (costs and times vary; the row numbers don't):

```
                                    QUERY PLAN
----------------------------------------------------------------------------------
 Seq Scan on orders  (cost=0.00..... rows=5 width=...) (actual time=... rows=6 loops=1)
   Filter: (customer_id = 42)
   Rows Removed by Filter: 99994
 Planning Time: ... ms
 Execution Time: ... ms
```

How to read it:
- `Seq Scan` = PostgreSQL read **every** row of `orders`.
- `rows=5` (the estimate — about 100,000 orders ÷ 20,000 customers; yours may say 4–6) vs `rows=6` (actual) — close, so the statistics are good.
- `Rows Removed by Filter: 99994` — it read 100,000 rows to keep 6. That ratio is the signature of a **missing index**.

`EXPLAIN` alone shows the estimated plan *without running* the query; `EXPLAIN ANALYZE` runs it and adds the actual numbers.

✅ **Checkpoint:** 6 rows returned, 99,994 removed.

### Step 2 — Add an index and watch the plan change

```sql
CREATE INDEX idx_orders_customer_id ON source_oltp.orders (customer_id);

EXPLAIN ANALYZE
SELECT * FROM source_oltp.orders WHERE customer_id = 42;
```

**Expected output** (one of these two shapes):

```
 Bitmap Heap Scan on orders  (cost=... rows=5 ...) (actual time=... rows=6 loops=1)
   Recheck Cond: (customer_id = 42)
   Heap Blocks: exact=...
   ->  Bitmap Index Scan on idx_orders_customer_id  (... rows=5 ...) (actual ... rows=6 loops=1)
         Index Cond: (customer_id = 42)
```

or `Index Scan using idx_orders_customer_id on orders … Index Cond: (customer_id = 42)`. Either way, `Rows Removed by Filter` is gone and the execution time drops sharply. Read nested plans **from the innermost (most indented) node outwards**: the Bitmap Index Scan finds the row locations, the Bitmap Heap Scan fetches them.

PostgreSQL indexes **primary keys automatically, but not foreign keys** — `orders.customer_id` had no index until now.

✅ **Checkpoint:** the plan names `idx_orders_customer_id`; `\di source_oltp.*` lists it.

### Step 3 — When an index can't help: low selectivity

```sql
CREATE INDEX idx_orders_status ON source_oltp.orders (status);

EXPLAIN SELECT COUNT(*) FROM source_oltp.orders WHERE status = 'completed';
```

**Expected output:** a plan containing `Seq Scan on orders` with `Filter: ((status)::text = 'completed'::text)` — **not** the new index.

93,986 of 100,000 orders are completed (94%). Hopping through an index to fetch almost every row is slower than reading the table straight through, and the planner knows it. Indexes help selective filters; for "most of the table" a Seq Scan is the *right* plan.

```sql
DROP INDEX source_oltp.idx_orders_status;   -- useless index: remove it
```

**Expected output:** `DROP INDEX`

✅ **Checkpoint:** you can explain why the planner ignored the index.

### Step 4 — Composite indexes, column order, and function-wrapped columns

A store dashboard filters by store **and** date range:

```sql
CREATE INDEX idx_orders_store_date ON source_oltp.orders (store_id, order_date);

EXPLAIN ANALYZE
SELECT COUNT(*) FROM source_oltp.orders
WHERE store_id = 3
  AND order_date >= DATE '2025-01-01' AND order_date < DATE '2025-02-01';

SELECT COUNT(*) FROM source_oltp.orders
WHERE store_id = 3
  AND order_date >= DATE '2025-01-01' AND order_date < DATE '2025-02-01';
```

**Expected output:** a plan using `idx_orders_store_date` (an `Index Only Scan` or `Bitmap Index Scan` with `Index Cond: ((store_id = 3) AND (order_date >= …) AND (order_date < …))`), then:

```
 count
-------
    68
(1 row)
```

**Column-order rule:** an index on `(a, b)` is sorted by `a`, then `b` — like a phone book sorted by last name, then first name. It serves filters on `a` or on `a AND b`, but not on `b` alone. Put **equality** columns first and **range** columns last.

Now wrap the column in a function:

```sql
EXPLAIN SELECT COUNT(*) FROM source_oltp.orders
WHERE store_id = 3 AND EXTRACT(YEAR FROM order_date) = 2025;
```

**Expected output:** the `Index Cond` only contains `(store_id = 3)`; the year test appears as a separate `Filter: (EXTRACT(year FROM order_date) = '2025'::numeric)`. The index stores `order_date`, not `EXTRACT(YEAR FROM order_date)`, so it cannot search by year. Rewrite such filters as ranges (`order_date >= '2025-01-01' AND order_date < '2026-01-01'`).

✅ **Checkpoint:** the count is 68 and you can say which part of the second query used the index.

### Step 5 — Diagnose the guided lab's slow report

Run the guided lab's Step 1.2 diagnosis (it adds `BUFFERS` — how many 8 KB pages came from memory (`shared hit`) vs disk (`read`)):

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    s.store_name,
    sr.region_name,
    pc.category_name,
    p.product_name,
    DATE_TRUNC('month', o.order_date) AS sales_month,
    COUNT(DISTINCT o.order_id)                            AS order_count,
    SUM(oi.quantity)                                      AS units_sold,
    SUM(oi.quantity * oi.unit_price)                      AS gross_revenue,
    SUM(oi.discount_amount)                               AS total_discounts,
    SUM(oi.quantity * oi.unit_price - oi.discount_amount) AS net_revenue
FROM source_oltp.orders             o
JOIN source_oltp.order_items        oi ON o.order_id       = oi.order_id
JOIN source_oltp.products           p  ON oi.product_id    = p.product_id
JOIN source_oltp.product_subcats    ps ON p.subcategory_id = ps.subcategory_id
JOIN source_oltp.product_categories pc ON ps.category_id   = pc.category_id
JOIN source_oltp.stores             s  ON o.store_id       = s.store_id
JOIN source_oltp.store_districts    sd ON s.district_id    = sd.district_id
JOIN source_oltp.store_regions      sr ON sd.region_id     = sr.region_id
WHERE o.order_date >= DATE '2025-01-01'
  AND o.order_date <  DATE '2026-01-01'
GROUP BY s.store_name, sr.region_name, pc.category_name,
         p.product_name, DATE_TRUNC('month', o.order_date)
ORDER BY sr.region_name, s.store_name, sales_month;
```

**Expected output** (shape — your timings differ): a tree whose top is `Sort` / `GroupAggregate` (`rows=122481` actual), below it a stack of `Hash Join` nodes, with `Seq Scan on order_items` (`rows=300235`) at the bottom, a scan of `orders` filtered to 2025, and `Buffers: shared hit=…` lines on each node. The last lines are `Planning Time` and `Execution Time`.

How to find the expensive part: a node's **own** time is its `actual time` total minus its children's. Here the big costs are scanning all of `order_items`, building the hash joins, and the sort/aggregate on five text columns. An index wouldn't fix that — the query genuinely needs a year of lines. **The fix is structural: pre-joined dimensions and integer keys (Labs 04–06) and pre-aggregation (Lab 08).**

✅ **Checkpoint:** you can point at the Seq Scan on `order_items` and the Sort/GroupAggregate in your plan.

### Step 6 — Statistics and write cost

```sql
ANALYZE source_oltp.orders;
SELECT relname, n_live_tup, last_analyze IS NOT NULL AS analyzed
FROM pg_stat_user_tables
WHERE schemaname = 'source_oltp' AND relname = 'orders';
```

**Expected output:**

```
ANALYZE
 relname | n_live_tup | analyzed
---------+------------+----------
 orders  |     100000 | t
(1 row)
```

Run `ANALYZE` after every big load (you'll do this in Lab 08 before tuning the warehouse); otherwise the planner reasons about yesterday's row counts. **Every index slows down every INSERT/UPDATE/DELETE**, because each index must be maintained too — index what your `WHERE` and `JOIN` clauses actually need, and prove each index with a before/after plan.

✅ **Checkpoint:** `analyzed` is `t`.

### Step 7 — Clean up the source

The guided lab expects the OLTP source *without* these extra indexes (its Step 1.2 diagnosis relies on that). Remove them:

```sql
DROP INDEX source_oltp.idx_orders_customer_id;
DROP INDEX source_oltp.idx_orders_store_date;
```

**Expected output:**

```
DROP INDEX
DROP INDEX
```

✅ **Checkpoint:** `\di source_oltp.*` lists only the primary-key / unique indexes plus `idx_promotion_products_product` and `uq_customer_primary_address`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Plan still shows `Seq Scan` after creating an index | Filter not selective, function-wrapped column, or stale statistics | Check selectivity; rewrite as a range; run `ANALYZE table;`. |
| Estimated `rows=` wildly different from actual | Stale statistics | `ANALYZE source_oltp.orders;` and re-run `EXPLAIN ANALYZE`. |
| `ERROR: relation "idx_orders_customer_id" already exists` | Step rerun | `DROP INDEX source_oltp.idx_orders_customer_id;` then recreate. |
| `EXPLAIN ANALYZE` on an `INSERT/UPDATE` changed data | `ANALYZE` really executes the statement | Wrap it: `BEGIN; EXPLAIN ANALYZE …; ROLLBACK;`. |
| Timings differ a lot between runs | Caching: first run reads from disk, later runs from memory | Compare the second run of each query; look at `shared hit` vs `read`. |

## Key takeaways

- `EXPLAIN ANALYZE` shows estimated vs actual rows and time per node; read the tree from the innermost node out.
- Indexes help selective filters and join keys; PostgreSQL does not index foreign keys for you.
- Composite index: equality columns first, range last; never wrap an indexed column in a function in `WHERE`.
- Some queries can't be indexed fast — they need a different data structure: the star schema.

## Next

[Lab 04 — Dimensional Modeling and the Warehouse DDL](Lab_04_Dimensional_Modeling_and_Warehouse_DDL.md): design and create that star schema.
