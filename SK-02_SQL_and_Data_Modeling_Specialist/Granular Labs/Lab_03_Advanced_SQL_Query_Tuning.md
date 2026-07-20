# Lab 03 — Advanced SQL & Query Tuning

> **Scenario:** FreshMart's dashboards take 40+ seconds to load, and the Monday-morning revenue report times out. You've been asked to "make the database faster." Time to learn what the database is actually *doing* — and how to change it.
>
> **Time:** 4–5 hours | **Difficulty:** Intermediate | **Prerequisite:** Labs 01–02

---

## 1. Environment Setup

Nothing new to install. Verify:

```powershell
psql -U postgres -d freshmart -c "SELECT COUNT(*) FROM order_lines;"
```

**Expected:** ~150,000. If this fails, see Lab 02 Section 1 for service/database troubleshooting.

Create the script file:

```powershell
New-Item -ItemType File -Path C:\dataeng\sk02\sql\03_tuning.sql -Force
```

One new configuration check — timing display in psql, so you see how long each query takes:

```
\timing on
```

**Expected output:** `Timing is on.` (pgAdmin shows timing automatically in the bottom-right of the Query Tool.)

---

## 2. Business Context

Slow queries cost real money three ways:

1. **People wait.** An analyst re-running a 40-second query 50 times a day loses half an hour daily.
2. **Systems fall over.** OLTP databases serving customers *and* heavy reports at once cause checkout timeouts — this is the core reason warehouses exist (Lab 04).
3. **Cloud bills explode.** On Snowflake/BigQuery/Redshift you literally pay per second of compute or per byte scanned. A tuned query is a cheaper query.

**Who consumes this skill:** every data engineer, every day. **What happens if it fails:** a common production incident is someone adding an innocent-looking query to a dashboard that does a full scan of the largest table every 30 seconds — and the whole database slows for everyone. The engineer who can read an execution plan finds it in minutes; the one who can't restarts servers and prays.

---

## 3. Concept Explanation

### The query planner

You write *declarative* SQL — **what** you want, not **how** to get it. PostgreSQL's **planner** chooses the how: which table to read first, which join algorithm, whether to use an index. It estimates costs using **statistics** (row counts, value distributions) it collects via `ANALYZE`.

### EXPLAIN and EXPLAIN ANALYZE

- `EXPLAIN <query>` — shows the *plan* with cost estimates, without running the query.
- `EXPLAIN ANALYZE <query>` — actually runs it and shows estimated **vs actual** rows and timing per node. The gap between estimate and actual is where the interesting bugs live.

Plans are trees read **innermost/deepest first**. Key node types:

| Node | Meaning | When it's fine / bad |
|---|---|---|
| `Seq Scan` | Read the whole table | Fine for small tables or reading most rows; bad when fetching a few rows from millions |
| `Index Scan` | Use an index, fetch matching rows | Great for selective filters |
| `Index Only Scan` | Answer entirely from the index | Best case — never touches the table |
| `Nested Loop` | For each outer row, probe inner side | Great when outer side is tiny |
| `Hash Join` | Build hash table of one side, probe with other | Workhorse for big joins |
| `Sort` / `HashAggregate` | ORDER BY / GROUP BY machinery | Watch for sorts spilling to disk |

### Indexes

An **index** is a separate, sorted data structure (a **B-tree** by default) mapping column values → row locations — like a book's index versus reading every page. Trade-offs:

- **Pro:** turns "scan 150,000 rows" into "hop to ~50 rows."
- **Con:** every INSERT/UPDATE/DELETE must also update every index — indexes tax writes. Also disk space.
- **Composite index** `(a, b)`: usable for filters on `a` or on `a AND b` — but *not* on `b` alone (like a phone book sorted by last name, then first: useless for finding all "Kwames").

**Alternatives to indexing:** rewrite the query, pre-aggregate (materialized views), partition the table, or accept the cost. Indexing is not always the answer — a report reading 90% of a table gains nothing from an index.

### Materialized views

A **view** is a saved query (runs fresh every time). A **materialized view** physically stores the result — instant reads, but **stale** until you `REFRESH` it. It's the simplest form of the pre-computation idea that entire warehouses are built on.

---

## 4. Step-by-Step Implementation

### Step 1 — Read your first plan

**What/why:** establish the baseline: a filtered lookup with no index.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
```

**Expected output (shape, numbers vary):**

```
Seq Scan on orders  (cost=0.00..1060.00 rows=25 width=29)
                    (actual time=0.02..5.8 rows=27 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 49973
Execution Time: 5.9 ms
```

**How to read it:** PostgreSQL scanned all 50,000 rows (`Seq Scan`), threw away 49,973 (`Rows Removed by Filter`), and kept ~27. That ratio — read everything to keep almost nothing — is the signature of a missing index.
**Verify:** `Rows Removed by Filter` + returned rows ≈ total table rows.
**Common mistake:** panicking that 5.9 ms is "slow." It isn't — on 50k rows even bad plans are fast. The *shape* is what scales badly: at 50M rows this becomes 6 seconds.
**Troubleshooting:** estimates wildly off from actuals (e.g., `rows=1` estimated, 20,000 actual)? Statistics are stale — run `ANALYZE orders;` and re-explain.

### Step 2 — Add an index and watch the plan flip

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
```

**Expected output:** the plan becomes `Bitmap Index Scan` or `Index Scan using idx_orders_customer_id`, execution time drops to well under 1 ms, and `Rows Removed by Filter` disappears.
**Verify:** `\di` in psql lists the index; the plan names it.
**Common mistake:** indexing and *not* re-checking the plan. The planner may still choose a Seq Scan if the filter isn't selective (e.g., `WHERE status = 'completed'` matches 94% of rows — the index would be slower, and PostgreSQL knows it).
**Troubleshooting:** planner refuses to use your index in a test? Force a comparison with `SET enable_seqscan = off;` (diagnostic only — **never** in production), compare, then `RESET enable_seqscan;`.

### Step 3 — Index the foreign keys that joins depend on

**What/why:** PostgreSQL indexes primary keys automatically, **not** foreign keys. Every join in Lab 02 walked `order_lines.order_id` — unindexed.

```sql
CREATE INDEX idx_order_lines_order_id   ON order_lines (order_id);
CREATE INDEX idx_order_lines_product_id ON order_lines (product_id);
CREATE INDEX idx_orders_store_id        ON orders (store_id);
CREATE INDEX idx_orders_order_ts        ON orders (order_ts);
```

**Expected output:** four `CREATE INDEX` confirmations.
**Verify:** re-run Lab 02 Step 7 (top 3 customers per store) with `EXPLAIN ANALYZE` before/after — join nodes shift and time drops (modestly at this size; dramatically at scale).
**Common mistake:** indexing every column "just in case." Each index slows every write. Rule: index what your WHERE and JOIN clauses actually use, and verify with plans.
**Troubleshooting:** `CREATE INDEX` on a big production table locks writes — the production-safe variant is `CREATE INDEX CONCURRENTLY` (slower, but non-blocking). Know this for interviews.

### Step 4 — A composite index and the column-order rule

**What/why:** the dashboard's hottest query filters store *and* time range together.

```sql
CREATE INDEX idx_orders_store_ts ON orders (store_id, order_ts);

EXPLAIN ANALYZE
SELECT COUNT(*) FROM orders
WHERE store_id = 3
  AND order_ts >= '2025-01-01' AND order_ts < '2025-02-01';
```

**Expected output:** an Index Scan (or Index Only Scan) on `idx_orders_store_ts`, sub-millisecond.
**Verify:** now try filtering **only** `order_ts` with `EXPLAIN` — the composite index is unlikely to be used (leading column missing); the standalone `idx_orders_order_ts` from Step 3 is.
**Rule of thumb:** equality columns first, range columns last in a composite index.
**Common mistake:** wrapping the indexed column in a function — `WHERE DATE(order_ts) = '2025-01-15'` cannot use the plain index. Rewrite as a range (`order_ts >= '2025-01-15' AND order_ts < '2025-01-16'`) or create an expression index.

### Step 5 — Find the expensive node in a real report

**What/why:** tune the actual monthly-revenue-by-region report from Lab 02.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT s.region, DATE_TRUNC('month', o.order_ts)::date AS month,
       SUM(ol.line_amount) AS revenue
FROM orders o
JOIN order_lines ol ON ol.order_id = o.order_id
JOIN stores s       ON s.store_id = o.store_id
WHERE o.status = 'completed'
GROUP BY 1, 2;
```

**Expected output:** a multi-node tree — likely Hash Joins feeding a HashAggregate. `BUFFERS` adds `shared hit/read` lines: pages served from RAM vs disk.
**How to find the expensive node:** look at each node's *own* time (its `actual time` total minus its children's). Here it's usually the `order_lines` scan — unavoidable, because the query genuinely needs every completed line. **Correct conclusion: this query doesn't need an index; it needs pre-computation.**
**Verify your reading:** total `Execution Time` at the bottom ≈ the root node's actual total time.
**Common mistake:** "there's a Seq Scan, add an index!" A report aggregating most of a table *should* Seq Scan — sequential reads of everything beat 150,000 random index hops.

### Step 6 — Materialized view for the expensive report

```sql
CREATE MATERIALIZED VIEW mv_monthly_region_revenue AS
SELECT s.region, DATE_TRUNC('month', o.order_ts)::date AS month,
       SUM(ol.line_amount) AS revenue,
       COUNT(DISTINCT o.order_id) AS orders
FROM orders o
JOIN order_lines ol ON ol.order_id = o.order_id
JOIN stores s       ON s.store_id = o.store_id
WHERE o.status = 'completed'
GROUP BY 1, 2;

-- The dashboard now reads this instead:
SELECT * FROM mv_monthly_region_revenue ORDER BY month, region;
```

**Expected output:** the SELECT returns ~72 rows (4 regions × ~18 months) in ~0 ms — versus hundreds of ms for the raw aggregation.
**Verify:** `EXPLAIN ANALYZE SELECT * FROM mv_monthly_region_revenue;` shows a tiny Seq Scan of 72 rows.
**Refresh strategy (the part interviews probe):** the MV is frozen at creation time. New orders won't appear until:

```sql
REFRESH MATERIALIZED VIEW mv_monthly_region_revenue;
```

In production you schedule this (Windows Task Scheduler + a `psql -f refresh.sql` command, or your orchestrator in SK-03) and document the staleness contract: *"data current as of last nightly refresh."* `REFRESH ... CONCURRENTLY` (requires a unique index on the MV) lets reads continue during refresh.
**Common mistake:** stakeholders assuming MV data is live. Always expose a freshness timestamp — you'll build exactly that into the ETL audit table in Lab 06.

### Step 7 — Measure index write cost, then clean up

**What/why:** prove the "indexes tax writes" claim.

```sql
\timing on
INSERT INTO orders (customer_id, store_id, order_ts, status)
SELECT 1 + (random()*1999)::int, 1 + (random()*7)::int, now(), 'completed'
FROM generate_series(1, 10000);
```

Note the time. Then drop the four secondary indexes on `orders`, repeat the insert, compare, and **recreate the indexes** (Labs 05–06 rely on them). Delete the test rows:

```sql
DELETE FROM orders WHERE order_ts >= now() - INTERVAL '10 minutes' AND order_id > 50000;
```

**Expected:** the indexed insert is measurably slower (often 1.5–3× at this scale).
**Verify:** `SELECT COUNT(*) FROM orders;` is back to ~50,000, and `\di` shows all indexes restored.
**Common mistake:** forgetting to recreate the indexes — Lab 05's incremental loads will crawl and you'll wrongly blame your ETL SQL.

Save everything to `03_tuning.sql`.

---

## 5. Production Engineering Practices

1. **Plan before and after, always.** Never claim a fix without an EXPLAIN ANALYZE diff. *Failure story:* an engineer added five indexes to "fix" a slow dashboard; the dashboard was slow because of a `DATE()` function wrapper the indexes couldn't serve anyway — but the nightly load, now paying five-index write tax, blew its window and reports were empty at 9 a.m.*
2. **Statistics hygiene.** Autovacuum usually handles `ANALYZE`, but after bulk loads (like ETL — Lab 05), run `ANALYZE <table>;` explicitly so the planner isn't reasoning about yesterday's row counts.
3. **Name indexes by convention.** `idx_<table>_<cols>` — so plans, monitoring, and migrations are readable.
4. **Document staleness contracts.** Any materialized/pre-computed data must state its refresh schedule where consumers can see it.
5. **Non-blocking operations in production.** `CREATE INDEX CONCURRENTLY`, `REFRESH MATERIALIZED VIEW CONCURRENTLY` — the pattern of "correct but blocking" vs "slower but online" recurs everywhere in data engineering.

---

## 6. Reflection

**What you learned:** reading execution plans, when indexes help (selective filters, join keys) and when they don't (broad aggregations), composite index column ordering, function-wrapped predicates, index write cost, and materialized views with refresh strategies.

**Why it matters:** performance tuning is where data engineers earn credibility — it's the difference between "the person who writes SQL" and "the person we call when the database is on fire."

### Interview questions

1. **EXPLAIN vs EXPLAIN ANALYZE?** *EXPLAIN shows the estimated plan without running; ANALYZE runs the query and shows actual rows/timings. Compare estimates vs actuals to spot stale statistics.*
2. **When does an index NOT help?** *Low-selectivity filters (matching most rows), queries aggregating whole tables, columns wrapped in functions, and tiny tables. And every index slows writes.*
3. **Why is a Seq Scan sometimes the right choice?** *Reading most of a table sequentially is cheaper than millions of random index probes. The planner costs both and picks the cheaper.*
4. **Explain composite index column order.** *The index is sorted by the first column, then the second. Filters must include the leading column(s) to use it; put equality predicates first, ranges last.*
5. **Does PostgreSQL index foreign keys automatically?** *No — only primary keys and unique constraints. Unindexed FKs are a top cause of slow joins and slow cascading deletes.*
6. **View vs materialized view?** *A view is a saved query, always fresh, no storage. A materialized view stores results — fast but stale until REFRESH; you must own a refresh schedule and communicate staleness.*
7. **A query got slow overnight with no code change. First three checks?** *(1) EXPLAIN ANALYZE — did the plan change? (2) Stale statistics — ANALYZE the tables. (3) Data growth or bloat — row counts, autovacuum health, locks.*
8. **What is index-only scan?** *The query is answered entirely from index data without visiting the table — possible when all selected columns are in the index and visibility permits.*

**Common trap:** "Would you add an index to speed up `SELECT * FROM huge_table`?" No — no filter, no help; that query's problem is `SELECT *` itself.

**Next:** [Lab 04 — Dimensional Modeling](Lab_04_Dimensional_Modeling.md). You've been tuning queries against an OLTP schema; now you'll design a schema where analytics is fast *by construction*.
