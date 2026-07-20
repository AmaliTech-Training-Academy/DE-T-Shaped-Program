# Lab 01: SQL Foundations — SELECT, Filtering, NULLs, and How Queries Actually Execute

> **Goal:** Build FreshMart's source (OLTP) tables, load realistic data with pure SQL, and master the core querying skills everything else stands on: SELECT, WHERE, ORDER BY, LIMIT, expressions, and the NULL rules that silently corrupt reports when misunderstood.
>
> **Time:** 3–4 hours
> **Prerequisites:** Lab 00 completed — PostgreSQL running, `freshmart` database exists

---

## 1. Environment Setup

Later labs only verify and add what's new — that starts now.

**Verify your environment (2 minutes):**

```powershell
Get-Service postgresql*        # Status must be Running
psql -U postgres -d freshmart -c "SELECT 1;"
```

Expected: service `Running`, and the query prints a column `?column?` with value `1`. If either fails, revisit Lab 00 Steps 3–5.

**New for this lab:** nothing to install. Create one file: `C:\dataeng\sk02\sql\01_create_oltp.sql` — you'll build it as you go. Save every script; Labs 04–06 reuse these tables.

**How to run scripts:** In pgAdmin's Query Tool use **Open File** then F5. In psql: `\i 'C:/dataeng/sk02/sql/01_create_oltp.sql'` (forward slashes).

---

## 2. Business Context

Meet **FreshMart**: a national grocery chain, 200 stores, ~15 million transactions a year. Its point-of-sale and e-commerce systems write into a PostgreSQL **OLTP** database.

**OLTP** (Online Transaction Processing) means a database optimized for many small, fast reads and writes — ring up a sale, update stock, register a customer. Its opposite, **OLAP** (Online Analytical Processing), means big read-heavy questions over history — "sales by region by month for 3 years." This module's whole arc is: learn to query the OLTP (Labs 01–03), then build a proper OLAP warehouse from it (Labs 04–06).

Right now FreshMart's analysts run reports directly on the OLTP. It's slow and it disrupts the stores — but before you can fix that (Lab 04+), you must be able to *query* it fluently. In this lab you play the role of a new data engineer told: "Here's the database. Get familiar. Finance has some questions."

- **Who consumes your output?** Analysts and finance staff who trust your numbers into board decks.
- **What happens if you get it wrong?** A misunderstood NULL or a bad filter produces a *plausible but wrong* number. Nobody notices for months. Then someone reconciles against the general ledger, and trust in the entire data team evaporates. Wrong-but-plausible is far more dangerous than an obvious crash.

---

## 3. Concept Explanation

### 3.1 Tables, rows, columns, types

A **table** is a named grid. Each **column** has a name and a **data type** (what kind of values it holds); each **row** is one record. Types you'll use constantly:

| Type | Holds | Example |
|---|---|---|
| `INT` / `BIGINT` | Whole numbers | `42` |
| `NUMERIC(10,2)` | Exact decimals — **always use for money** | `19.99` |
| `TEXT` / `VARCHAR(n)` | Strings | `'Accra'` |
| `DATE` | Calendar date | `'2026-01-31'` |
| `TIMESTAMPTZ` | Instant in time with time zone | `'2026-01-31 14:03:00+00'` |
| `BOOLEAN` | `TRUE` / `FALSE` / `NULL` | `TRUE` |

**Why NUMERIC for money, not FLOAT?** `FLOAT` is binary floating point: `0.1 + 0.2` gives `0.30000000000000004`. Over millions of transactions those errors accumulate and your totals won't reconcile with finance. `NUMERIC` is exact. Real incident category: companies have failed audits over float-summed money.

### 3.2 SQL statement families

- **DDL** (Data Definition Language): defines structure — `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`.
- **DML** (Data Manipulation Language): changes data — `INSERT`, `UPDATE`, `DELETE`.
- **Queries**: `SELECT` — reads data, changes nothing.

### 3.3 Logical execution order — the single most useful mental model in SQL

You *write* a query in this order, but the database *evaluates* it in a different order:

```
Written:   SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
Executed:  FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

Consequences you'll hit within the hour:
- A column alias created in `SELECT` **cannot** be used in `WHERE` (WHERE runs first, the alias doesn't exist yet).
- It **can** be used in `ORDER BY` (runs after SELECT).
- `WHERE` filters rows *before* grouping; `HAVING` filters *after* (Lab 02).

Memorize the executed order. It explains 80% of beginner SQL errors.

### 3.4 NULL — the three-valued logic trap

**NULL means "unknown/absent," not zero and not empty string.** SQL comparisons involving NULL return neither TRUE nor FALSE but **UNKNOWN**, and WHERE only keeps rows where the condition is TRUE. So:

- `NULL = NULL` → UNKNOWN (not TRUE!). Use `IS NULL`.
- `email <> 'x@y.com'` **excludes rows where email is NULL** — they're neither equal nor not-equal, they're unknown.
- `COALESCE(a, b, c)` returns the first non-NULL argument — your main repair tool.
- `a IS DISTINCT FROM b` is a NULL-safe "not equal": treats two NULLs as *same*, NULL vs value as *different*. This becomes critical in SCD change detection in Lab 05 — remember it.

**Alternatives?** Some teams ban NULLs via `NOT NULL` + defaults. Good where possible; but real source data always has genuinely-unknown values, so you must handle NULLs correctly regardless.

---

## 4. Step-by-Step Implementation

Work in pgAdmin's Query Tool connected to `freshmart` (or psql after `\c freshmart`). Save everything into `01_create_oltp.sql` as you go.

### Step 1 — Create the FreshMart OLTP tables

**What/why:** We create a small but realistic slice of FreshMart's operational schema: customers, stores, products, orders, and order lines. This is the source system every later lab reads from.

```sql
-- FreshMart OLTP schema (simplified slice)
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    full_name     TEXT        NOT NULL,
    email         TEXT,                        -- nullable: in-store customers may not give one
    city          TEXT        NOT NULL,
    segment       TEXT        NOT NULL DEFAULT 'Standard',  -- Standard | Loyalty | Premium
    signup_date   DATE        NOT NULL
);

CREATE TABLE stores (
    store_id      SERIAL PRIMARY KEY,
    store_name    TEXT NOT NULL,
    region        TEXT NOT NULL                -- North | South | East | West
);

CREATE TABLE products (
    product_id    SERIAL PRIMARY KEY,
    product_name  TEXT          NOT NULL,
    category      TEXT          NOT NULL,      -- Produce | Dairy | Bakery | Beverages | Household
    unit_price    NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0)
);

CREATE TABLE orders (
    order_id      SERIAL PRIMARY KEY,
    customer_id   INT  NOT NULL REFERENCES customers(customer_id),
    store_id      INT  NOT NULL REFERENCES stores(store_id),
    order_ts      TIMESTAMPTZ NOT NULL,
    status        TEXT NOT NULL DEFAULT 'completed'   -- completed | cancelled | returned
);

CREATE TABLE order_lines (
    order_line_id SERIAL PRIMARY KEY,
    order_id      INT NOT NULL REFERENCES orders(order_id),
    product_id    INT NOT NULL REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    line_amount   NUMERIC(12,2) NOT NULL       -- quantity * price after discounts
);
```

**New concepts in that DDL:**
- `PRIMARY KEY` — a column that uniquely identifies each row; the database enforces uniqueness and disallows NULL.
- `SERIAL` — auto-incrementing integer; the database assigns 1, 2, 3… so you don't have to.
- `REFERENCES` — a **foreign key**: `orders.customer_id` must match an existing `customers.customer_id`. This is *referential integrity* — the database refuses orphan orders.
- `CHECK` — a rule every row must satisfy. Cheap insurance against garbage (negative prices).
- `DEFAULT` — value used when the INSERT doesn't supply one.

**Expected output:** five `CREATE TABLE` confirmations.
**Verify:** `\dt` in psql (or refresh pgAdmin's Tables node) shows all 5 tables.
**Common mistake:** creating tables while connected to `postgres` instead of `freshmart`. Check the prompt/connection bar first.
**Troubleshooting:** `relation "customers" already exists` → you ran it twice. `DROP TABLE order_lines, orders, products, stores, customers;` and rerun (drop children before parents — foreign keys enforce order).

### Step 2 — Load data with pure SQL (GENERATE_SERIES)

**What/why:** Rather than hand-typing rows or hunting CSVs, we *generate* realistic volume with `generate_series()` — a PostgreSQL function that produces a sequence of numbers as rows. Data engineers use this trick constantly for testing, and you'll reuse it for `dim_date` in Lab 05.

```sql
-- 8 stores
INSERT INTO stores (store_name, region) VALUES
 ('Accra Central','South'),('Kumasi Main','North'),('Tema Harbour','East'),
 ('Takoradi Mall','West'),('Accra East','South'),('Tamale Plaza','North'),
 ('Cape Coast','West'),('Ho Market','East');

-- 25 products across 5 categories
INSERT INTO products (product_name, category, unit_price)
SELECT
    'Product ' || gs,
    (ARRAY['Produce','Dairy','Bakery','Beverages','Household'])[1 + (gs % 5)],
    ROUND((2 + random() * 48)::numeric, 2)
FROM generate_series(1, 25) AS gs;

-- 2,000 customers; ~15% have no email (NULL); mixed segments
INSERT INTO customers (full_name, email, city, segment, signup_date)
SELECT
    'Customer ' || gs,
    CASE WHEN random() < 0.15 THEN NULL
         ELSE 'customer' || gs || '@example.com' END,
    (ARRAY['Accra','Kumasi','Tema','Takoradi','Tamale'])[1 + (gs % 5)],
    CASE WHEN random() < 0.10 THEN 'Premium'
         WHEN random() < 0.40 THEN 'Loyalty'
         ELSE 'Standard' END,
    DATE '2023-01-01' + (random() * 900)::int
FROM generate_series(1, 2000) AS gs;

-- 50,000 orders over ~18 months; ~4% cancelled, ~2% returned
INSERT INTO orders (customer_id, store_id, order_ts, status)
SELECT
    1 + (random() * 1999)::int,
    1 + (random() * 7)::int,
    TIMESTAMPTZ '2024-06-01' + (random() * 540) * INTERVAL '1 day'
        + (random() * 86400) * INTERVAL '1 second',
    CASE WHEN random() < 0.04 THEN 'cancelled'
         WHEN random() < 0.06 THEN 'returned'
         ELSE 'completed' END
FROM generate_series(1, 50000);

-- ~150,000 order lines: 1–5 lines per order
INSERT INTO order_lines (order_id, product_id, quantity, line_amount)
SELECT
    o.order_id,
    p.product_id,
    q.quantity,
    ROUND(q.quantity * p.unit_price, 2)
FROM orders o
CROSS JOIN LATERAL (
    SELECT 1 + (random() * 4)::int AS n_lines
) AS lines
CROSS JOIN LATERAL (
    SELECT 1 + (random() * 24)::int AS product_id,
           1 + (random() * 3)::int  AS quantity,
           gs
    FROM generate_series(1, lines.n_lines) AS gs
) AS q
JOIN products p ON p.product_id = q.product_id;
```

**Don't panic at the last statement** — `CROSS JOIN LATERAL` is an advanced construct meaning "for each order, run this mini-subquery." You are *using* it as a data generator, not being tested on it. By Lab 03 you'll read it comfortably.

**Expected output:** `INSERT 0 8`, `INSERT 0 25`, `INSERT 0 2000`, `INSERT 0 50000`, and roughly `INSERT 0 150000` (random, ~120k–160k).

**Verify:**

```sql
SELECT
  (SELECT COUNT(*) FROM customers)   AS customers,
  (SELECT COUNT(*) FROM orders)      AS orders,
  (SELECT COUNT(*) FROM order_lines) AS order_lines;
```

Expected: `2000 | 50000 | ~150000`.

**Note:** `random()` means your exact numbers will differ from this lab's examples — that's fine and intentional. Check *shapes* (row counts, orders of magnitude), not exact values.

**Troubleshooting:** foreign-key violation on the orders insert → your customers insert didn't complete; recheck counts table by table.

### Step 3 — Your first real SELECTs

**What/why:** `SELECT` is how you read. Start simple and layer clauses one at a time.

```sql
-- 3a. Everything from a small table (fine for 8 rows, never for millions)
SELECT * FROM stores;

-- 3b. Specific columns, renamed with aliases
SELECT store_name AS store, region FROM stores;

-- 3c. Peek at a big table SAFELY: always LIMIT when exploring
SELECT * FROM orders LIMIT 10;
```

**Why avoid `SELECT *` in real code?** It fetches columns you don't need (wasted I/O), and it breaks downstream code when someone adds a column. Fine for interactive peeking; banned in production pipelines.

**Verify:** 3a returns 8 rows; 3c returns exactly 10.
**Common mistake:** forgetting the `;` in psql — you get the `-#` continuation prompt. Type `;` Enter.

### Step 4 — Filtering with WHERE

```sql
-- Completed orders at store 1 in January 2025
SELECT order_id, order_ts, status
FROM orders
WHERE status = 'completed'
  AND store_id = 1
  AND order_ts >= '2025-01-01'
  AND order_ts <  '2025-02-01'
ORDER BY order_ts
LIMIT 20;
```

**Why `>= start AND < next-month` instead of `BETWEEN`?** `BETWEEN '2025-01-01' AND '2025-01-31'` on a *timestamp* silently drops almost all of Jan 31 (everything after midnight 00:00:00). The half-open range `>= / <` is always safe for dates and timestamps. This is a classic production bug: month-end numbers mysteriously low.

Other WHERE tools to try now:

```sql
SELECT * FROM products WHERE category IN ('Dairy','Bakery');           -- set membership
SELECT * FROM products WHERE unit_price BETWEEN 10 AND 20;             -- fine for numbers
SELECT * FROM customers WHERE full_name LIKE 'Customer 19%' LIMIT 5;   -- pattern match, % = anything
SELECT * FROM customers WHERE city ILIKE 'accra';                      -- ILIKE = case-insensitive (Postgres-specific)
```

**Verify:** each returns plausible rows; the IN query returns only the two categories.
**Common mistake:** single vs double quotes. In SQL, **strings use single quotes** `'text'`; double quotes `"name"` are for identifiers (column/table names). `WHERE city = "Accra"` errors with `column "Accra" does not exist`.

### Step 5 — Feel the NULL trap on real data

```sql
-- How many customers have no email?
SELECT COUNT(*) FROM customers WHERE email IS NULL;      -- correct: ~300 (15% of 2000)
SELECT COUNT(*) FROM customers WHERE email = NULL;       -- WRONG: always 0!
```

Run both. The second returns **0** even though ~300 rows have NULL email — because `= NULL` evaluates to UNKNOWN for every row. Now the subtler one:

```sql
-- "Customers whose email is not at example.com" — looks right, silently wrong:
SELECT COUNT(*) FROM customers WHERE email NOT LIKE '%@example.com';   -- 0, and misses NULLs anyway

-- NULL-safe version:
SELECT COUNT(*) FROM customers
WHERE email IS NULL OR email NOT LIKE '%@example.com';

-- Repair tool: COALESCE for display
SELECT full_name, COALESCE(email, '(no email on file)') AS contact
FROM customers LIMIT 10;

-- NULL-safe comparison (memorize for Lab 05!)
SELECT NULL = NULL AS naive, NULL IS NOT DISTINCT FROM NULL AS null_safe;
-- naive: (null)   null_safe: true
```

**Why this matters:** imagine "count customers we can't email" for a marketing budget. `= NULL` returns 0; marketing buys no postal campaign; 300 customers are never contacted. Wrong-but-plausible.

### Step 6 — Expressions, CASE, and date functions

```sql
SELECT
    order_id,
    order_ts,
    order_ts::date                              AS order_date,      -- :: casts type
    EXTRACT(YEAR FROM order_ts)                 AS order_year,
    TO_CHAR(order_ts, 'YYYY-MM')                AS order_month,     -- format as text
    CASE status
        WHEN 'completed' THEN 'Revenue'
        WHEN 'returned'  THEN 'Refund risk'
        ELSE 'No revenue'
    END                                         AS finance_view
FROM orders
LIMIT 15;
```

- `::date` — Postgres cast syntax (standard equivalent: `CAST(order_ts AS date)`).
- `EXTRACT` — pull a part out of a date/timestamp.
- `CASE` — SQL's if/else; you'll use it in almost every analytical query.

**Verify:** `order_month` looks like `2025-03`; `finance_view` maps statuses correctly.

Now use the alias-in-WHERE rule from §3.3 — try to break it:

```sql
SELECT order_id, EXTRACT(YEAR FROM order_ts) AS order_year
FROM orders
WHERE order_year = 2025;    -- ERROR: column "order_year" does not exist
```

**Why:** WHERE executes before SELECT; the alias doesn't exist yet. Fix by repeating the expression in WHERE (or use a subquery/CTE — Lab 02):

```sql
WHERE EXTRACT(YEAR FROM order_ts) = 2025
```

### Step 7 — Sorting and pagination

```sql
-- Most expensive products first; ties broken by name for deterministic order
SELECT product_name, category, unit_price
FROM products
ORDER BY unit_price DESC, product_name ASC
LIMIT 10;
```

**Why the tie-breaker?** Without it, rows with equal prices come back in *arbitrary* order — your "top 10" can differ between runs, which makes reports non-reproducible. Production rule: **ORDER BY must fully determine order** whenever results feed a report. Note NULLs sort *last* in ascending order by default in Postgres (`NULLS FIRST/LAST` overrides).

### Step 8 — Save your work as a script and rerun it

Save all of Steps 1–2 into `C:\dataeng\sk02\sql\01_create_oltp.sql`, but make it **rerunnable** by putting drops at the top:

```sql
DROP TABLE IF EXISTS order_lines, orders, products, stores, customers;
-- ... then all CREATE TABLE and INSERT statements ...
```

**Why `IF EXISTS` and why drops first?** So the script can run on a fresh database *or* over an existing one, always ending in the same state. That property is called **idempotency** — running a process twice yields the same result as once. It is arguably the single most important production ETL property, and this is your first taste of it.

**Verify:** run the whole script twice in a row. Second run must succeed with no errors and the count query from Step 2 must still show the right shapes.

---

## 5. Production Engineering Practices

1. **Idempotent scripts (Step 8).** *Failure story:* a team's setup script had bare `CREATE TABLE`s. It failed halfway on a rerun, leaving half-old half-new tables; the following ETL loaded into the stale half. Two days of "impossible" numbers. `DROP IF EXISTS` + rerun-from-clean would have prevented it.
2. **Half-open date ranges (Step 4).** `>= start AND < end` — never `BETWEEN` on timestamps. Real bug class: month-end revenue understated by one day, discovered at audit.
3. **NUMERIC for money (Step 1).** Float rounding errors don't reconcile with finance.
4. **Deterministic ORDER BY (Step 7).** Reports must be reproducible run-to-run.
5. **Constraints as your first data-quality layer (Step 1).** `NOT NULL`, `CHECK`, foreign keys reject garbage at the door instead of letting it flow downstream. Lab 06 expands this into a full quality framework.
6. **Numbered, saved scripts.** `01_create_oltp.sql` in a versioned folder — your future self, and Git in real jobs, need the history.

---

## 6. Reflection

### What you learned
DDL and constraints; generating test data at volume with `generate_series`; SELECT/WHERE/ORDER BY/LIMIT; casting, `EXTRACT`, `CASE`; the logical execution order; and NULL three-valued logic including `IS DISTINCT FROM`.

### Why it matters
Every analytical query, every ETL job, and Labs 02–06 compose these primitives. The NULL rules and execution order in particular are where wrong-but-plausible numbers come from.

### Interview questions

1. **"What order does SQL execute a query's clauses in?"** — FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT. That's why SELECT aliases work in ORDER BY but not WHERE.
2. **"Why does `WHERE col = NULL` return no rows?"** — Any comparison with NULL yields UNKNOWN, and WHERE keeps only TRUE. Use `IS NULL` / `IS NOT NULL`.
3. **"What's the difference between WHERE and HAVING?"** — WHERE filters rows before grouping; HAVING filters groups after aggregation. (Full practice in Lab 02.)
4. **"How would you filter a timestamp column for 'January 2025' safely?"** — `ts >= '2025-01-01' AND ts < '2025-02-01'`. BETWEEN on timestamps drops most of the last day.
5. **"Why NUMERIC instead of FLOAT for money?"** — FLOAT is inexact binary; sums drift and fail reconciliation. NUMERIC is exact decimal.
6. **"What is a foreign key and what does it protect against?"** — A constraint requiring a value to exist in another table's key; prevents orphan rows (an order for a customer that doesn't exist).
7. **"What does idempotent mean for a SQL script?"** — Running it N times produces the same end state as once, e.g. `DROP TABLE IF EXISTS` + CREATE + load, or `INSERT ... ON CONFLICT`.
8. **"What does `IS DISTINCT FROM` do?"** — NULL-safe inequality: NULL vs NULL is *not distinct* (false), NULL vs value is distinct (true). Essential for change detection in SCD loads.

**Common traps:** "`COUNT(column)` vs `COUNT(*)`" — `COUNT(column)` skips NULLs, `COUNT(*)` counts all rows; interviewers love this. Also `DELETE` vs `TRUNCATE` vs `DROP`: removes rows (filterable, logged) / removes all rows fast / removes the table itself.

### Key takeaways
- Execution order explains most "weird" SQL errors.
- NULL is unknown; guard every comparison and remember `IS DISTINCT FROM`.
- Constraints and idempotent scripts are production habits from day one.
- FreshMart's OLTP is now loaded — **Lab 02** teaches you to combine and aggregate it.
