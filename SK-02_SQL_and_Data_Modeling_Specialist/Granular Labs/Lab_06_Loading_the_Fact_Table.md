# Lab 06 — Loading the Fact Table

> **Goal:** Load `fact_sales` incrementally and idempotently with SCD-aware surrogate-key lookups, reconcile it against the source, then simulate a day of source changes and watch SCD Type 2 preserve history.
> **Time:** 150–180 min | **Difficulty:** Intermediate | **Prerequisites:** Lab 05
> **Guided lab coverage:** PART 4 (Steps 4.1–4.3) of `SK-02_End_to_End_Guided_Lab.md`

## Learning objectives

By the end of this lab you can:

- Write PL/pgSQL `DO $$ … $$` blocks with `DECLARE`, `SELECT … INTO`, `IF`, `RETURN`, `RAISE NOTICE` and `GET DIAGNOSTICS`.
- Explain and implement a **high-water mark** (HWM) incremental load with a frozen upper bound.
- Look up surrogate keys with **SCD-aware date-range joins** (`order_date BETWEEN effective_start AND effective_end`).
- Use `COALESCE(…, -1)` for the placeholder promotion and `ON CONFLICT (grain key) DO UPDATE` for idempotency.
- **Reconcile** source and target (row counts and money), and diagnose a mismatch.
- Run the nightly chain after source changes and verify that history is preserved.

**Key terms**

| Term | Meaning |
|---|---|
| **High-water mark** | The latest source `updated_at` already loaded; the next run loads only rows changed after it. |
| **Incremental load** | Loading only new/changed rows instead of reloading everything. |
| **Reconciliation** | Comparing source and target totals after a load. "No error" is not proof of success; matching totals are. |
| **Late-arriving dimension** | A fact whose dimension version doesn't exist (yet) for its date — an inner join drops it silently. |

## Steps

### Step 1 — PL/pgSQL DO blocks

SQL statements can't hold variables or `IF`s. A `DO $$ … $$` block runs a small anonymous PL/pgSQL program — and the whole block runs as **one transaction**.

```sql
DO $$
DECLARE
    v_completed BIGINT;
    v_rows      BIGINT;
BEGIN
    SELECT COUNT(*) INTO v_completed
    FROM source_oltp.orders
    WHERE status = 'completed';

    IF v_completed > 0 THEN
        RAISE NOTICE 'Completed orders: %', v_completed;   -- % is replaced by the value
    END IF;

    UPDATE etl_control SET high_water_mark = high_water_mark WHERE table_name = 'fact_sales';
    GET DIAGNOSTICS v_rows = ROW_COUNT;                    -- rows affected by the last statement
    RAISE NOTICE 'etl_control rows touched: %', v_rows;
END $$;
```

**Expected output:**

```
NOTICE:  Completed orders: 93986
NOTICE:  etl_control rows touched: 1
DO
```

(`$$` is just a quote mark for the block's body, so you don't have to escape the `'` inside it.)

✅ **Checkpoint:** both notices appear.

### Step 2 — The high-water mark and the SCD-aware lookup, by hand

```sql
-- Where did the last run stop? (nothing loaded yet)
SELECT high_water_mark FROM etl_control WHERE table_name = 'fact_sales';

-- Where will this run stop? Frozen BEFORE loading, so rows arriving
-- during the load are picked up next time instead of being skipped.
SELECT MAX(updated_at) AS new_hwm
FROM source_oltp.orders
WHERE updated_at > TIMESTAMP '1970-01-01';

-- Which customer VERSION was valid on the date of order 13616 (customer 42, 2025-12-31)?
SELECT dc.customer_key, dc.segment, dc.effective_start, dc.effective_end
FROM dim_customer dc
WHERE dc.customer_id = 42
  AND DATE '2025-12-31' BETWEEN dc.effective_start AND dc.effective_end;
```

**Expected output:**

```
   high_water_mark
---------------------
 1970-01-01 00:00:00
(1 row)

       new_hwm
---------------------
 2025-12-31 23:56:01
(1 row)

 customer_key | segment  | effective_start | effective_end
--------------+----------+-----------------+---------------
           42 | Standard | 1900-01-01      | 9999-12-31
(1 row)
```

The fact load does exactly this for every line: join `dim_customer` and `dim_product` on **natural key + date range** (never on `is_current = TRUE`, which would rewrite history to today's attributes), `dim_store` on `store_id` (Type 1), and `dim_promotion` on the promotion actually applied to the line — `COALESCE(promotion_key, -1)` when there was none.

✅ **Checkpoint:** the lookup returns exactly one version.

### Step 3 — Run the fact load (guided Step 4.1)

Read the script's comments: HWM read → upper bound frozen → `RETURN` early if nothing is new → one `INSERT … SELECT` with the SCD-aware joins, `status = 'completed'` business rule and measure calculations → `ON CONFLICT (order_id, order_line_number) DO UPDATE` → `GET DIAGNOSTICS` → HWM advanced in the **same** transaction → reconciliation query.

Save as `C:\DataEngineering\sk02\sql\08_load_fact_sales.sql`:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 08_load_fact_sales.sql — incremental, idempotent fact_sales load
-- High-water mark = MAX(source orders.updated_at) already loaded,
-- stored in etl_control. SCD-aware joins use the order date.
-- A DO block runs as ONE transaction: load + HWM advance are atomic.
-- ═══════════════════════════════════════════════════════════════

DO $$
DECLARE
    v_hwm      TIMESTAMP;   -- where the last run stopped
    v_new_hwm  TIMESTAMP;   -- where this run will stop
    v_rows     BIGINT;
BEGIN
    SELECT high_water_mark INTO v_hwm
    FROM   etl_control
    WHERE  table_name = 'fact_sales';

    -- Freeze the upper bound BEFORE loading, so rows arriving during the
    -- load are picked up next time instead of being skipped.
    SELECT MAX(updated_at) INTO v_new_hwm
    FROM   source_oltp.orders
    WHERE  updated_at > v_hwm;

    RAISE NOTICE 'Loading fact_sales: updated_at in (%, %]', v_hwm, v_new_hwm;

    IF v_new_hwm IS NULL THEN
        RAISE NOTICE 'Nothing new to load.';
        RETURN;
    END IF;

    INSERT INTO fact_sales (
        date_key, customer_key, product_key, store_key, promotion_key,
        order_id, order_line_number,
        quantity, unit_price, unit_cost, discount_amount,
        gross_revenue, net_revenue, gross_margin
    )
    SELECT
        TO_CHAR(o.order_date, 'YYYYMMDD')::INT          AS date_key,
        dc.customer_key,                                -- version valid on the order date
        dp.product_key,                                 -- version valid on the order date
        ds.store_key,
        COALESCE(dpr.promotion_key, -1)                 AS promotion_key,
        o.order_id,
        oi.line_number                                  AS order_line_number,
        oi.quantity,
        oi.unit_price,
        dp.unit_cost,                                   -- cost as it was at time of sale
        oi.discount_amount,
        oi.quantity * oi.unit_price                     AS gross_revenue,
        oi.quantity * oi.unit_price - oi.discount_amount AS net_revenue,
        oi.quantity * oi.unit_price - oi.discount_amount
            - oi.quantity * dp.unit_cost                AS gross_margin
    FROM source_oltp.orders       o
    JOIN source_oltp.order_items  oi ON o.order_id = oi.order_id

    -- SCD-aware dimension lookups using the order date
    JOIN dim_customer dc
        ON  o.customer_id = dc.customer_id
        AND o.order_date  BETWEEN dc.effective_start AND dc.effective_end
    JOIN dim_product dp
        ON  oi.product_id = dp.product_id
        AND o.order_date  BETWEEN dp.effective_start AND dp.effective_end
    JOIN dim_store ds
        ON  o.store_id    = ds.store_id
    -- The promotion actually applied to the line (at most one → no fan-out)
    LEFT JOIN dim_promotion dpr
        ON  oi.promotion_id = dpr.promotion_id

    WHERE o.updated_at >  v_hwm
      AND o.updated_at <= v_new_hwm
      AND o.status = 'completed'            -- business rule: revenue = completed orders

    -- Idempotency: a line already in the fact is updated, never duplicated
    ON CONFLICT (order_id, order_line_number) DO UPDATE SET
        date_key        = EXCLUDED.date_key,
        customer_key    = EXCLUDED.customer_key,
        product_key     = EXCLUDED.product_key,
        store_key       = EXCLUDED.store_key,
        promotion_key   = EXCLUDED.promotion_key,
        quantity        = EXCLUDED.quantity,
        unit_price      = EXCLUDED.unit_price,
        unit_cost       = EXCLUDED.unit_cost,
        discount_amount = EXCLUDED.discount_amount,
        gross_revenue   = EXCLUDED.gross_revenue,
        net_revenue     = EXCLUDED.net_revenue,
        gross_margin    = EXCLUDED.gross_margin,
        etl_loaded_at   = CURRENT_TIMESTAMP;

    GET DIAGNOSTICS v_rows = ROW_COUNT;

    UPDATE etl_control
    SET    high_water_mark = v_new_hwm
    WHERE  table_name = 'fact_sales';

    RAISE NOTICE 'Upserted % rows into fact_sales; high-water mark now %', v_rows, v_new_hwm;
END $$;

-- Reconciliation: source (completed lines) vs warehouse — must match exactly
SELECT
    (SELECT COUNT(*)
       FROM source_oltp.order_items oi
       JOIN source_oltp.orders o ON o.order_id = oi.order_id
      WHERE o.status = 'completed')                                  AS source_lines,
    (SELECT COUNT(*) FROM fact_sales)                                AS fact_lines,
    (SELECT SUM(oi.quantity * oi.unit_price - oi.discount_amount)
       FROM source_oltp.order_items oi
       JOIN source_oltp.orders o ON o.order_id = oi.order_id
      WHERE o.status = 'completed')                                  AS source_net_revenue,
    (SELECT SUM(net_revenue) FROM fact_sales)                        AS fact_net_revenue;
```

Run it (≈30–60 s):

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/08_load_fact_sales.sql
```

**Expected output:**

```
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Loading fact_sales: updated_at in (1970-01-01 00:00:00, 2025-12-31 23:56:01]
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Upserted 282259 rows into fact_sales; high-water mark now 2025-12-31 23:56:01
DO
 source_lines | fact_lines | source_net_revenue | fact_net_revenue
--------------+------------+--------------------+------------------
       282259 |     282259 |        17315871.37 |      17315871.37
(1 row)
```

282,259 = the order lines of the 93,986 completed orders (of 300,235 lines in total).

✅ **Checkpoint:** the reconciliation row matches column for column.

### Step 4 — Check the measures and the placeholder promotion

```sql
SELECT COUNT(*)                                       AS fact_rows,
       SUM(gross_revenue)                             AS gross_revenue,
       SUM(discount_amount)                           AS discounts,
       SUM(net_revenue)                               AS net_revenue,
       SUM(gross_margin)                              AS gross_margin,
       COUNT(*) FILTER (WHERE promotion_key = -1)     AS lines_without_promotion
FROM fact_sales;
```

**Expected output:**

```
 fact_rows | gross_revenue | discounts | net_revenue | gross_margin | lines_without_promotion
-----------+---------------+-----------+-------------+--------------+-------------------------
    282259 |   17410472.45 |  94601.08 | 17315871.37 |   5559012.37 |                  275009
(1 row)
```

`gross_revenue − discounts = net_revenue` (17410472.45 − 94601.08). 7,250 lines had a promotion; all the others point at the `-1` "No Promotion" row instead of holding NULL.

✅ **Checkpoint:** the subtraction checks out.

### Step 5 — Prove it is incremental and idempotent

Run `08_load_fact_sales.sql` again.

**Expected output:**

```
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Loading fact_sales: updated_at in (2025-12-31 23:56:01, <NULL>]
psql:/sql/08_load_fact_sales.sql:96: NOTICE:  Nothing new to load.
DO
 source_lines | fact_lines | source_net_revenue | fact_net_revenue
--------------+------------+--------------------+------------------
       282259 |     282259 |        17315871.37 |      17315871.37
(1 row)
```

That proves **incrementality** (the HWM skips old rows). Now prove **idempotency** — that even reprocessing everything can't duplicate a line. Rewind the mark and rerun:

```sql
UPDATE etl_control SET high_water_mark = '1970-01-01' WHERE table_name = 'fact_sales';
```

Then run `08_load_fact_sales.sql` once more. **Expected:** `NOTICE: Upserted 282259 rows …` — every line was *updated*, not inserted — and the reconciliation still shows `282259 | 282259`. The `UNIQUE (order_id, order_line_number)` constraint plus `ON CONFLICT DO UPDATE` make a double-load impossible.

✅ **Checkpoint:** `fact_lines` is still 282259 after the rewound run.

### Step 6 — A day passes: source changes (guided Step 4.2)

Save as `C:\DataEngineering\sk02\sql\09_simulate_day2_changes.sql` and run it **once**:

```sql
-- ═══════════════════════════════════════════════════════════════
-- 09_simulate_day2_changes.sql — "a day passes in the OLTP system"
-- Run ONCE, after the first full ETL run. Then rerun scripts 05–08.
-- NOT idempotent on purpose (it is a test fixture, not ETL).
-- ═══════════════════════════════════════════════════════════════

BEGIN;

-- 1. 40 customers change segment (Premium ↔ Standard) → SCD2 on dim_customer
UPDATE source_oltp.customers
SET segment_id = CASE WHEN segment_id = 1 THEN 2 ELSE 1 END,
    updated_at = now()
WHERE customer_id % 500 = 0;

-- 2. 20 customers move their primary address to Denver → SCD2 on dim_customer
UPDATE source_oltp.customer_addresses
SET city = 'Denver', state = 'CO', postal_code = '80202'
WHERE is_primary AND customer_id % 1000 = 7 AND city <> 'Denver';

-- 3. 12 products get a 10% price increase → SCD2 on dim_product
UPDATE source_oltp.products
SET list_price = round(list_price * 1.10, 2),
    updated_at = now()
WHERE product_id % 100 = 1;          -- products 1, 101, 201, ... 1101

-- 4. Product 7 is recategorised (Yogurt → Salads, i.e. Dairy → Produce) → SCD2 on dim_product
UPDATE source_oltp.products
SET subcategory_id = 4, updated_at = now()
WHERE product_id = 7;

-- 5. Store 1 gets a new manager → Type 1 overwrite on dim_store
UPDATE source_oltp.stores
SET manager_id = 62, updated_at = now()
WHERE store_id = 1;

-- 6. A new order arrives today for customer 500 (who just changed segment)
INSERT INTO source_oltp.orders (customer_id, store_id, order_date, status, updated_at)
VALUES (500, 1, CURRENT_DATE, 'completed', now());

INSERT INTO source_oltp.order_items (order_id, line_number, product_id, quantity, unit_price, discount_amount)
SELECT currval('source_oltp.orders_order_id_seq'), l.line_number, l.product_id, 2, p.list_price, 0
FROM (VALUES (1, 101), (2, 7)) AS l(line_number, product_id)
JOIN source_oltp.products p ON p.product_id = l.product_id;

COMMIT;
```

**Expected output:**

```
BEGIN
UPDATE 40
UPDATE 17
UPDATE 12
UPDATE 1
UPDATE 1
INSERT 0 1
INSERT 0 2
COMMIT
```

(`UPDATE 17`, not 20: three of the twenty customers already lived in Denver.)

Now run the nightly chain in dependency order — dimensions first, facts last: `05_load_type1_dims.sql`, `06_scd2_dim_customer.sql`, `07_scd2_dim_product.sql`, `08_load_fact_sales.sql`.

**Expected output (key lines):**

| Script | You should see |
|---|---|
| 05 | `INSERT 0 60`, `INSERT 0 40`, `60 \| 41` |
| 06 | `SELECT 20000`, `UPDATE 57`, `INSERT 0 57`, `UPDATE 0`, totals `20057 \| 20000` |
| 07 | `SELECT 1200`, `UPDATE 13`, `INSERT 0 13`, `UPDATE 0`, totals `1213 \| 1200` |
| 08 | `Upserted 2 rows into fact_sales; high-water mark now <today's timestamp>`; reconciliation `282261 \| 282261 \| 17316002.29 \| 17316002.29` |

57 = 40 segment changes + 17 moves; 13 = 12 price changes + product 7's recategorisation.

✅ **Checkpoint:** 20057 customer rows but still 20000 current ones; 2 new fact rows.

### Step 7 — Verify that history is preserved (guided Step 4.3)

```sql
SELECT customer_key, segment, city, effective_start, effective_end, is_current
FROM dim_customer WHERE customer_id = 500 ORDER BY effective_start;

SELECT dc.segment, COUNT(*) AS lines, SUM(fs.net_revenue) AS net_revenue
FROM fact_sales fs
JOIN dim_customer dc ON fs.customer_key = dc.customer_key
WHERE dc.customer_id = 500
GROUP BY dc.segment ORDER BY dc.segment;

SELECT dp.product_key, dp.subcategory_name, dp.category_name, dp.is_current,
       COUNT(fs.sales_key) AS lines, SUM(fs.net_revenue) AS net_revenue
FROM dim_product dp
LEFT JOIN fact_sales fs ON fs.product_key = dp.product_key
WHERE dp.product_id = 7
GROUP BY dp.product_key, dp.subcategory_name, dp.category_name, dp.is_current
ORDER BY dp.product_key;

SELECT store_id, store_name, manager_name FROM dim_store WHERE store_id = 1;
```

**Expected output** (`<yesterday>` / `<today>` are your run dates):

```
 customer_key | segment |   city    | effective_start | effective_end | is_current
--------------+---------+-----------+-----------------+---------------+------------
          500 | Budget  | Charlotte | 1900-01-01      | <yesterday>   | f
        20002 | Premium | Charlotte | <today>         | 9999-12-31    | t
(2 rows)

 segment | lines | net_revenue
---------+-------+-------------
 Budget  |    12 |      717.39
 Premium |     2 |      130.92
(2 rows)

 product_key | subcategory_name | category_name | is_current | lines | net_revenue
-------------+------------------+---------------+------------+-------+-------------
           7 | Yogurt           | Dairy         | f          |   229 |    20325.04
        1202 | Salads           | Produce       | t          |     1 |       69.76
(2 rows)

 store_id | store_name  | manager_name
----------+-------------+---------------
        1 | New York #1 | John Williams
(1 row)
```

Read it as a business user: customer 500's 12 historical lines stay **Budget** forever; today's order counts as **Premium**. Product 7's 229 historical sales stay in **Dairy**; today's sale is in **Produce**. Store 1 is Type 1 — the old manager ("Mary Davis") is simply gone.

✅ **Checkpoint:** two versions each for customer 500 and product 7, with facts split exactly as shown.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `fact_lines` lower than `source_lines` (even 0) | Inner joins dropped lines with no dimension version for their date — e.g. first versions starting at `CURRENT_DATE` instead of `1900-01-01`, or dimensions not loaded | Fix the dimension load (Lab 05), reset the mark (`UPDATE etl_control SET high_water_mark = '1970-01-01' …`) and rerun 08 — `ON CONFLICT` makes that safe. |
| `insert or update on table "fact_sales" violates foreign key constraint "fact_sales_date_key_fkey"` | `dim_date` empty or doesn't cover the order date | Run `04_load_dim_date.sql`. |
| `ON CONFLICT DO UPDATE command cannot affect row a second time` | A dimension join matched two versions (overlapping effective dates) | Run the one-current checks from Lab 05; never edit effective dates by hand. |
| `Nothing new to load.` on the very first run | `etl_control` already advanced | `UPDATE etl_control SET high_water_mark = '1970-01-01' WHERE table_name = 'fact_sales';` |
| `UPDATE 0` from 06/07 after the day-2 script | 09 not run, or run in another database | Check `SELECT segment_id FROM source_oltp.customers WHERE customer_id = 500;` (should be 1). |
| Step 7 shows today's order under the *old* versions (Budget / Dairy) | 09 ran before midnight, the nightly chain after it — the new versions start on a later day than the order | Run 09 and scripts 05–08 on the same calendar day (or rebuild and repeat). |
| Running 09 twice | It is a test fixture, not ETL | Rebuild from scratch: 01, 02, 03, 04, 05, 06, 07, 08, then 09 once. |

## Key takeaways

- A `DO` block gives SQL variables, control flow and messages — and runs as one transaction, so load + HWM advance are atomic.
- HWM = incremental; `UNIQUE` grain key + `ON CONFLICT` = idempotent. They are separate guarantees — you need both.
- Dimension lookups use natural key + date range; `COALESCE(key, -1)` for optional dimensions.
- Always reconcile counts *and* money between source and target after a load.

## Next

[Lab 07 — Analytical SQL and Window Functions](Lab_07_Analytical_SQL_and_Window_Functions.md): answer FreshMart's business questions on the star schema.
