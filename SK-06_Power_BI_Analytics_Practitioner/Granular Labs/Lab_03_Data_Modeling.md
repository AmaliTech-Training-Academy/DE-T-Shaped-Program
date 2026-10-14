# Lab 03 — Data Modeling: Star Schema & Relationships

> **Goal:** Turn the seven tables from Lab 02 into the guided lab's star schema — eight deliberate relationships (two of them inactive), a marked fiscal date table, correct sort orders and a tidy field list.
> **Time:** 4–5 hours | **Difficulty:** Intermediate | **Prerequisites:** Lab 02 (`NorthStar_Lab02.pbix` with seven tables and the row counts from Lab 02 Step 8)
> **Guided lab coverage:** PART 1 Step 1.1 (star schema design) and PART 2 Step 2.4 (relationships, date table, sort order); PART 6 Optimization 5
> **Produces:** `C:\DataEngineering\sk06\pbix\NorthStar_Lab03.pbix`

## Learning objectives

By the end of this lab you can:

- Tell facts from dimensions and state the **grain** of each fact table in one sentence.
- Explain why a star schema beats both one flat table and the normalised OLTP design for Power BI.
- Create one-to-many, single-direction relationships and explain how filters flow along them.
- Explain **active vs inactive** relationships and why NorthStar needs two inactive ones.
- Mark a date table, fix "alphabetical month" sorting, and switch off accidental summarisation of number attributes.

**Key terms**

| Term | Meaning |
|---|---|
| **Fact table** | Records events at a declared grain; mostly keys and numbers; long and narrow. You **aggregate** facts. |
| **Dimension table** | Describes context (who, what, where, when); short and wide. You **filter and group by** dimensions. |
| **Grain** | What one fact row represents. Mixing grains double-counts. |
| **Star schema** | Dimensions around facts, each one hop away. Kimball's design, and the shape VertiPaq and DAX are optimised for. |
| **Relationship** | A standing rule that filters flow from the **one** side (dimension) to the **many** side (fact). Not a join you write per query. |
| **Cardinality** | One-to-many (`1:*`) is the healthy norm. Many-to-many usually means a dimension key is not unique. |
| **Cross-filter direction** | **Single** (dimension → fact) by default. **Both** creates ambiguity, slows queries and can leak data past security. |
| **Active / inactive** | Only one relationship between two tables can be active. Others exist but sleep until a measure wakes them with `USERELATIONSHIP` (Lab 05). |
| **Mark as date table** | Tells DAX which table is the calendar; required for reliable time intelligence. |

### The NorthStar star (guided lab, Part 1)

```
                    dim_date
                      │ DateKey (PK)
                      │ ← fact_sales[OrderDateKey]      [ACTIVE]
                      │ ← fact_sales[ShipDateKey]       [INACTIVE]
                      │ ← fact_targets[target_date_key] [INACTIVE]
                      ↓
dim_customer ──→ fact_sales ←── dim_product
dim_store ────→  fact_sales ←── dim_promotion
dim_store ────→  fact_targets
```

**Grain statements** (write these into your notes — they are the most important sentences in the model):

- `fact_sales`: **one row per order line item** (310,024 rows).
- `fact_targets`: **one row per store per month**, stamped on the 1st of the month (3,060 rows = 85 stores × 36 months).

**Why not keep the OLTP tables?** In Lab 01 a region filter had to walk order_lines → stores → districts → regions. In the star, `dim_store` already contains `region_name`, so it is one hop. **Why not one big flat table?** Every store, product and customer attribute would repeat on 310,024 rows (poor compression), and the targets — a different grain — could not live in it at all. Facts never relate directly to facts; they share dimensions.

## Steps

### Step 1 — Open a copy and remove guessed relationships

1. `docker compose start` in `C:\DataEngineering\sk06` (needed for any refresh).
2. Open `NorthStar_Lab02.pbix` → **File → Save As** → `C:\DataEngineering\sk06\pbix\NorthStar_Lab03.pbix`.
3. Open **Model view** (third icon on the left). Power BI may have **auto-detected** relationships by matching column names (for example `store_id`).
4. **Home → Manage relationships** → select every listed relationship → **Delete** → **Close**.

**Why delete them?** An unexplained relationship is an unexplained filter path. Rebuilding eight relationships deliberately takes ten minutes; auditing guesses takes longer.

✅ **Checkpoint:** *Manage relationships* lists nothing.

### Step 2 — Relate the dimensions to fact_sales

**Home → Manage relationships → New**. For each row: pick the fact table and column at the top, the dimension and column at the bottom, then check that the dialog shows **Cardinality: Many to one (\*:1)** and **Cross filter direction: Single**, and **Make this relationship active** is ticked → **OK**.

| From (many side) | To (one side) |
|---|---|
| `fact_sales[OrderDateKey]` | `dim_date[DateKey]` |
| `fact_sales[store_id]` | `dim_store[store_id]` |
| `fact_sales[product_id]` | `dim_product[product_id]` |
| `fact_sales[customer_id]` | `dim_customer[customer_id]` |
| `fact_sales[promotion_id]` | `dim_promotion[promotion_id]` |

**If the dialog proposes Many to many — stop.** It means the dimension column is not unique (for example a merge in Lab 02 duplicated rows). Check the dimension's row count against Lab 02 Step 8. Never accept many-to-many to "make the error go away".

✅ **Checkpoint:** five solid lines into `fact_sales`, each with `1` at the dimension end and `*` at the fact end.

### Step 3 — Add the second date: ship date (inactive)

**Manage relationships → New**: `fact_sales[ShipDateKey]` → `dim_date[DateKey]`, Many to one, Single → **OK**.

Power BI makes it **inactive** automatically (a **dashed** line) because `fact_sales` and `dim_date` already have an active relationship. If it were active too, a date filter would not know whether to filter by order date or ship date — an ambiguous path.

`dim_date` is now a **role-playing dimension**: the same table plays "order date" (active, used by every measure) and "ship date" (inactive, used by the logistics measure `Revenue by Ship Date` in Lab 05). The ~1% of lines that never shipped have a blank `ShipDateKey` and simply match no date.

✅ **Checkpoint:** two lines between `fact_sales` and `dim_date` — one solid, one dashed.

### Step 4 — Relate fact_targets

| From (many side) | To (one side) | Active? |
|---|---|---|
| `fact_targets[store_id]` | `dim_store[store_id]` | Yes |
| `fact_targets[target_date_key]` | `dim_date[DateKey]` | **No** — untick *Make this relationship active* |

The guided lab makes the target date relationship inactive on purpose: the only measure that reads targets (`Revenue Target`, Lab 05) switches it on with `USERELATIONSHIP`, so date filtering of targets is explicit and lives in one measure.

✅ **Checkpoint:** *Manage relationships* lists **8** relationships; exactly **2** have the *Active* box unticked (`ShipDateKey` and `target_date_key`).

### Step 5 — Prove the filters flow (and that inactive means inactive)

Go to **Report view** and build a **Matrix**:

- Rows: `dim_store[region_name]`
- Values: `fact_sales[net_revenue]` and `fact_targets[revenue_target]` (both show as *Sum of …*)

**Expected output** (all three fiscal years):

| region_name | Sum of net_revenue | Sum of revenue_target |
|---|---|---|
| Midwest | 8,246,675.98 | 8,813,500.00 |
| Northeast | 9,308,148.76 | 9,024,500.00 |
| Southeast | 8,442,877.88 | 8,689,300.00 |
| West | 6,606,453.98 | 7,533,600.00 |
| **Total** | **32,604,156.60** | **34,060,900.00** |

Different values per region = the `dim_store` relationships work for **both** facts.

Now add a **Slicer** with `dim_date[Fiscal Year]` and pick **2026**:

- `Sum of net_revenue` changes: Northeast **3,346,490.01**, total **11,862,350.26** (your SQL control total from Lab 00).
- `Sum of revenue_target` does **not** change — its date relationship is inactive, so the year filter cannot reach `fact_targets`. That is expected; Lab 05's `Revenue Target` measure fixes it. (If the targets *do* change, the relationship is active — revisit Step 4.)

If every region shows the same `net_revenue`, the relationship is missing or uses the wrong column.

✅ **Checkpoint:** FY2026 Northeast net revenue 3,346,490.01; targets unaffected by the Fiscal Year slicer.

### Step 6 — Mark the date table

1. Data pane → select `dim_date` → **Table tools → Mark as date table** → choose the column **`Date`** → **OK**. Power BI validates that `Date` is unique, has no blanks and covers whole days.
2. While here, check the slicer from Step 5 — the numbers do not change. Marking changes how DAX time-intelligence functions treat the table (Lab 05), not the data.

**Why it matters:** `DATESYTD`, `SAMEPERIODLASTYEAR` and friends need a contiguous calendar column they can trust. Combined with Auto date/time off (Lab 00), `dim_date` is now the model's single calendar — Optimization 5 of the guided lab's Part 6.

✅ **Checkpoint:** the `dim_date` icon in the Data pane shows the date-table marker, and *Mark as date table* reports `Date` as the date column.

### Step 7 — Sort months properly (and see why one sort fails)

Text columns sort alphabetically by default ("April, August, December…"). Each text column can be sorted by **one** other column — and only if that other column has exactly one value for each text value.

1. Try the wrong one first: select `dim_date[Month Name]` → **Column tools → Sort by column → Year-Month Sort**.
   **Expected:** an error like *"There can't be more than one value in 'Year-Month Sort' for the same value in 'Month Name'"*. "July" occurs in many years, each with a different Year-Month Sort. This is exactly why the guided lab's trend chart uses `Year-Month` on the axis.
2. Now set the correct sorts:

| Column | Sort by column | Why |
|---|---|---|
| `Year-Month` | `Year-Month Sort` | 1:1 — "2025-07" ↔ 202507; the trend axis |
| `Month Name` | `Fiscal Month Number` | NorthStar reads July first |
| `Month Short` | `Fiscal Month Number` | same, short labels |
| `Day Name` | `Day of Week` | Monday first |

3. **Stop number attributes from being summed.** Select each of `dim_date[Year]`, `[Fiscal Year]`, `[Month Number]`, `[Fiscal Month Number]`, `[Week Number]`, `[Day of Week]`, `[DateKey]`, `[Year-Month Sort]` → **Column tools → Summarization → Don't summarize**. Otherwise dropping `Fiscal Year` into a table shows one meaningless *Sum of Fiscal Year* (millions) instead of the values 2024, 2025, 2026 … .

**Verify:** a table visual with `dim_date[Month Name]` lists **July, August, … June**.

✅ **Checkpoint:** months appear in fiscal order, and `Fiscal Year` in a table lists its values (2020 … 2028 — `dim_date` covers 2020–2027, and July–December 2027 belong to FY2028), not a sum.

### Step 8 — Hide the plumbing and set data categories

1. In **Model view**, hide (eye icon → *Hide in report view*) every key column on the facts: `fact_sales[OrderDateKey]`, `[ShipDateKey]`, `[store_id]`, `[product_id]`, `[customer_id]`, `[promotion_id]`, and `fact_targets[store_id]`, `[target_date_key]`. Report builders should pick attributes from dimensions — the dimension end of each relationship.
2. Also hide `dim_date[DateKey]` and `dim_date[Year-Month Sort]` (helpers, not for visuals).
3. `dim_store`: select `latitude` → **Column tools → Data category → Latitude**; `longitude` → **Longitude**; `city` → **City**; `state` → **State or Province**. Maps in Lab 06 need these.
4. Format money: select `fact_sales[net_revenue]`, `[gross_revenue]`, `[total_cost]`, `[gross_margin]` and `fact_targets[revenue_target]` → **Column tools → Format → Currency**, 2 decimals.
5. Arrange Model view as a star: facts in the middle, dimensions around. Future maintainers read this diagram.
6. **Ctrl+S**.

✅ **Checkpoint:** the Data pane no longer shows key columns on the facts; Model view looks like the diagram at the top of this lab.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Dialog offers only *Many to many* | Duplicate keys in the dimension | Check row counts (Lab 02 Step 8); fix the query that duplicated rows; never accept m:m here. |
| *You can't create a relationship between these two columns because one of the columns must have unique values* | You picked two fact columns, or reversed the tables | The **one** side must be the dimension key (`dim_*`). |
| Every region shows the same number | Relationship missing, inactive, or on the wrong column | Model view: the line `dim_store` → `fact_sales` must be solid and use `store_id`. |
| New ship-date relationship becomes active and the order-date one inactive | Order of creation / ticked *active* | Manage relationships: tick *Active* for `OrderDateKey`, untick for `ShipDateKey`. |
| *Mark as date table* fails validation | `Date` column not unique or has blanks | `dim_date` must come from Lab 02's query unchanged (2,922 rows). |
| Sort by column error on `Month Name` | Sorting by a column with several values per month name | Use `Fiscal Month Number` (or `Month Number`), not `Year-Month Sort`. |
| Visual shows *Sum of Fiscal Year* | Default summarization on a number attribute | Column tools → Summarization → Don't summarize. |
| Targets change with the Fiscal Year slicer | `target_date_key` relationship is active | Untick *Active* for it — Lab 05 activates it inside the measure. |

## Key takeaways

- Model shape decides everything downstream: most "DAX problems" are model problems.
- Filters flow from the one side to the many side along single-direction relationships. Internalise that arrow and DAX (Lab 04) becomes learnable.
- NorthStar's star: five dimensions, two facts at different grains, sharing `dim_store` and `dim_date`. Two inactive relationships (ship date, target date) wait for `USERELATIONSHIP`.
- `dim_date` is marked as the date table; month text columns sort by `Fiscal Month Number`, and `Year-Month` (sorted by `Year-Month Sort`) is the axis for trends.
- Hide keys, set data categories and formats: a tidy field list is part of the product.

**Interview check:** *Power BI proposes a many-to-many relationship — what do you do?* — Treat it as a smell: the dimension key is not unique. Find and fix the duplicates; genuine many-to-many needs a deliberate bridge table. *How do you report the same fact by two different dates?* — One active relationship (order date) and one inactive (ship date) to the same date table, and a measure using `USERELATIONSHIP` for the second role.

## Next

[Lab 04 — DAX Fundamentals](Lab_04_DAX_Fundamentals.md): the guided lab's base measures, filter context and CALCULATE — checked against your SQL control totals.
