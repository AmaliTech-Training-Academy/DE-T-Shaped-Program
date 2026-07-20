# Lab 04 — DAX Fundamentals: Measures, Filter Context & CALCULATE

> **Module:** SK-06 Power BI Analytics Practitioner
> **Estimated time:** 7–8 hours
> **Builds on:** Lab_03 — open `NorthStar_Lab03.pbix`, **Save As** `NorthStar_Lab04.pbix`.
> **Produces:** `NorthStar_Lab04.pbix` with a first measure library (~12 measures) in a dedicated measure table.

DAX (Data Analysis Expressions) is the calculation language of Power BI, Analysis Services, and Fabric semantic models. It looks like Excel formulas and behaves like nothing you've used before — because every formula's answer depends on **where it is evaluated**. This lab teaches that evaluation model properly, because memorizing snippets without it is how people stay stuck at "intermediate" for years.

---

## 1. Environment Setup

1. Open `NorthStar_Lab03.pbix` → Save As `C:\PowerBI_Labs\NorthStar\pbix\NorthStar_Lab04.pbix`.
2. Verify the Lab_03 model: star schema intact, `dim_date` marked as date table, fiscal columns verified, hidden foreign keys.
3. **Recommended (optional): DAX Studio** — free, from `daxstudio.org` → Downloads → installer → next-next-finish. It appears under **External Tools**. Used here only to peek at query behavior; everything works without it.
4. New-ish Power BI versions include **DAX query view** (left edge). If you have it, you can test measures with `EVALUATE` queries without building visuals — handy but optional.

---

## 2. Business Context

Lab_01 Failure 2: there was no drag-and-drop path to *revenue*, because revenue is logic, not a column. Every business runs on such defined numbers — revenue, margin, average order value, return rate — and the definitions have owners, edge cases, and consequences.

The business value of DAX measures is **centralized definitions**: "Net Revenue" is written once, in one place, with one owner — and every visual, every report, every user gets the same number. Compare that to NorthStar's status quo, where revenue is re-derived in dozens of spreadsheet cells, each a chance to diverge. In the capstone, Daniel makes this a contractual requirement: 15+ measures, each with a written business definition (FR-09) — because his analysts' definitions of "turnover" had drifted apart and clients noticed.

**Who consumes measures?** Everyone downstream — plus auditors and interviewers. **If measures are wrong**, they're wrong on every page simultaneously; centralization concentrates risk as well as truth. That's why this lab ends with validation against control numbers.

---

## 3. Concept Explanation

### 3.1 Measures vs calculated columns (the first fork in the road)

| | Calculated column | Measure |
|---|---|---|
| Computed | Once, at refresh — stored per row | At query time — never stored |
| Evaluated in | **Row context** (current row) | **Filter context** (current filters) |
| Cost | Disk/RAM (model size) | CPU (query time) |
| Use for | Row-level attributes to slice/filter by | Anything you aggregate |

Rule: **if it aggregates, it's a measure.** Columns are for attributes (and even then, prefer Power Query, as in Lab_02's `net_amount` — computed upstream, compresses at load).

### 3.2 Filter context — the concept that explains everything

A measure has no fixed answer. In a matrix of regions by month, the *same* measure is evaluated once per cell, and each cell first assembles a **filter context** — the set of filters coming from the visual's rows/columns, slicers, page filters, and cross-highlights — then applies it to the model (filters flowing 1→many along Lab_03's relationships), and only then evaluates the arithmetic over the surviving rows.

`Total Quantity` in the West/July cell = sum of quantity over *only* fact rows reachable from West and July. In the grand total cell, filter context is nearly empty, so it sums everything. **The formula never changed; the context did.** Once this clicks, DAX stops being magic.

### 3.3 Row context — the other context

Inside a calculated column, or inside an **iterator** function, evaluation walks a table row by row; "the current row" is the **row context**. Crucially, row context does **not** filter the model, and filter context does not give you a current row. Confusing the two is the root of ~80% of beginner DAX errors.

**Iterators** (`SUMX`, `AVERAGEX`, `MAXX`, ...) create row context on demand: `SUMX(table, expression)` evaluates the expression per row of `table` (itself already reduced by the filter context), then sums. This is how sum-of-products is done — Lab_01's revenue failure, solved.

### 3.4 CALCULATE — the filter-context editor

`CALCULATE(expression, filter1, filter2, ...)` evaluates the expression **under a modified filter context**: each filter argument overrides or adds to the incoming filters. It is the single most important function in DAX and the only common way to *change* context rather than accept it.

```dax
CALCULATE ( [Net Revenue], dim_store[region] = "West" )         -- force West
CALCULATE ( [Net Revenue], ALL ( dim_store ) )                  -- remove store filters (e.g., for % of total)
CALCULATE ( [Net Revenue], KEEPFILTERS ( dim_product[category] = "Toys" ) )  -- intersect, don't override
```

`ALL` removes filters; `REMOVEFILTERS` is its modern, clearer alias for that use. Every "% of total", "share of parent", time-intelligence, and RLS-adjacent pattern is CALCULATE with different filter arguments.

Alternatives to DAX? Precompute aggregates upstream (SK-03-style) — fine for fixed grains, but the whole point of a semantic model is answering *any* slice at query time; only a context-aware language does that. MDX (the older cube language) and LookML solve the same problem in other stacks.

---

## 4. Step-by-Step Implementation

### Step 1 — Create a measure table

Measures technically live "in" a table; scattering them across fact tables makes them hard to find. Convention: a dedicated, empty measure table.

1. **Home → Enter data** → leave the single empty column, name the table `_Measures` → Load. (The underscore sorts it to the top of the Data pane.)
2. After creating your first measure in it (Step 2), hide the table's dummy column; the table icon turns into a calculator glyph.

**Why:** discoverability. In a 60-measure model, "where is that measure" must never be a question. (Capstone NFR-05 checks organization.)

### Step 2 — First measures: the explicit basics

Right-click `_Measures` → **New measure**, one at a time:

```dax
Total Quantity = SUM ( fact_sales[quantity] )
```

```dax
Net Revenue = SUM ( fact_sales[net_amount] )
```

```dax
Order Count = DISTINCTCOUNT ( fact_sales[order_id] )
```

Format each immediately (**Measure tools** ribbon): `Net Revenue` currency 0 decimals; counts whole numbers with thousands separators. *Formatting at creation is a habit — unformatted measures look broken to stakeholders and get "fixed" inconsistently per visual.*

**Verification:** card with `Net Revenue` — a plausible total (tens of millions given 50k lines). Matrix rows `dim_store[region]`, values all three measures — four distinct rows plus a total. Note you dragged **the measure**, not a column: no `Σ` ambiguity, one authored definition.

**Common mistake:** *New measure* vs *New column* on the right-click menu are adjacent. If your "measure" appears as a column with rows of values in Table view, you made a column — delete and redo. Measures never appear in Table view.

### Step 3 — Prove you understand filter context (5-minute exercise)

Build a matrix: rows `dim_product[category]`, columns `dim_date[Fiscal Year]`, values `Net Revenue`. For any one inner cell, say out loud what its filter context is ("category = X AND fiscal year = FYnnnn, flowing to fact_sales via two relationships"). Then explain the column-total cell (year only) and the grand total (no filters). If you can narrate all three, you understand filter context better than most working analysts. Keep this matrix on the page — it's your test harness for the rest of the lab.

### Step 4 — The iterator you've been waiting for

Lab_02 pre-computed `net_amount` in Power Query, which is best practice. But you must also be able to do it in DAX — sources aren't always yours to edit:

```dax
Net Revenue (X) =
SUMX (
    fact_sales,
    fact_sales[quantity] * fact_sales[unit_price] * ( 1 - fact_sales[discount_pct] )
)
```

How it evaluates in a West/FY2026 cell: filter context reduces `fact_sales` to West+FY2026 rows → SUMX walks those rows (row context), computes the product per row → sums. Filter context *then* row context, in that order, every time.

**Verification:** `Net Revenue (X)` equals `Net Revenue` in every cell of your test matrix. If they diverge, your Power Query `net_amount` formula and this one disagree (check the discount handling).

```dax
Avg Order Value =
DIVIDE ( [Net Revenue], [Order Count] )
```

**Always `DIVIDE`, never `/`:** DIVIDE returns blank (or a chosen alternate) on divide-by-zero instead of an error that kills the visual. Blank cells are fine; error dialogs in front of a CFO are not.

### Step 5 — CALCULATE: returns, shares, and rates

Lab_02 flagged returns; now put the flag to work:

```dax
Returned Quantity =
CALCULATE ( ABS ( [Total Quantity] ), fact_sales[is_return] = TRUE () )
```

```dax
Sold Quantity =
CALCULATE ( [Total Quantity], fact_sales[is_return] = FALSE () )
```

```dax
Return Rate % =
DIVIDE ( [Returned Quantity], [Sold Quantity] )
```

Format `Return Rate %` as percentage, 1 decimal. **Verification:** ~1% globally (the Lab_00 injection rate). Slice by category — variation is expected.

Now the classic **% of total**, which is CALCULATE+ALL:

```dax
Revenue % of All Regions =
DIVIDE (
    [Net Revenue],
    CALCULATE ( [Net Revenue], REMOVEFILTERS ( dim_store ) )
)
```

In a West cell: numerator sees West; denominator *removes* the store filters, seeing all regions. **Verification:** region matrix — the four percentages sum to 100%; the total row shows 100%. Then add a `Fiscal Year` slicer and select one year: percentages still sum to 100% *within the year*, because `REMOVEFILTERS(dim_store)` removed only store filters, leaving date filters intact. That selectivity is the entire art of CALCULATE.

**Common mistake:** `REMOVEFILTERS()` with no argument removes *everything* (dates too) — the % then answers a different business question. Be surgical.

### Step 6 — Filter with FILTER when the condition needs a row

Boolean filter arguments (`col = value`) only work on single columns. Conditions comparing columns or using measures need `FILTER`:

```dax
Big Ticket Revenue =
CALCULATE (
    [Net Revenue],
    FILTER ( fact_sales, fact_sales[quantity] * fact_sales[unit_price] >= 500 )
)
```

`FILTER(table, condition)` is itself an iterator (row context!) returning the surviving rows, which CALCULATE then applies as a filter. Prefer boolean arguments when possible (they're faster); reach for FILTER when the condition genuinely needs row-by-row logic.

### Step 7 — Actual vs target (two facts, one context)

Lab_03's conformed `dim_region` earns its keep:

```dax
Target Revenue = SUM ( fact_targets[target_revenue] )
```

```dax
Revenue vs Target % =
DIVIDE ( [Net Revenue], [Target Revenue] )
```

**Verification:** matrix rows `dim_region[region]`, columns `dim_date[Month]` (sorted by Month Sort — Lab_03), values all three. Both facts populate every cell because both are filtered through the *shared* dimensions. If `Target Revenue` repeats one value everywhere, its relationships to `dim_region`/`dim_date` are missing or inactive — Model view, check the lines.

### Step 8 — Validate against control numbers, then save

1. In PowerShell, compute an independent control total:

   ```powershell
   $s = Import-Csv C:\PowerBI_Labs\NorthStar\data\sales.csv
   ($s | Measure-Object -Sum quantity).Sum
   ```

2. Compare to the `Total Quantity` card (no filters). They must match exactly. Repeat mentally for one region if you're feeling thorough.
3. Document each measure: select it → **Measure tools** → add a **Description** (right-click in Model view → Properties → Description): one plain-English business definition each. This is Daniel's FR-09 habit, started now.
4. Save `NorthStar_Lab04.pbix`.

---

## 5. Production Engineering Practices

- **A measure library is an API.** Named, formatted, documented, organized (`_Measures`, display folders as it grows). Consumers build on it without reading its internals — exactly like a code library from SK-01. Breaking a measure definition is a breaking API change: version notes matter.
- **Validation against independent control totals.** Never ship a measure you haven't reconciled against a number computed *outside* the model. *Failure story:* a retailer's new "Net Sales" measure silently included returns as positives (an ABS in the wrong place). It overstated revenue ~2% for a quarter before an accountant reconciling to the GL caught it. A one-line PowerShell control total on day one would have, too.
- **DIVIDE, formats, descriptions — defaults of professionals.** Each is 10 seconds at creation and an incident when skipped.
- **Measure once, reference everywhere.** Build compound measures from base measures (`[Net Revenue]`, not a re-typed SUM) so a definition change propagates. You did this in every ratio above.
- **Prefer upstream computation** for row-level logic (Power Query column) and DAX for context-dependent aggregation — cost model: storage vs query CPU.

---

## 6. Reflection

### What you learned

Measures vs columns; filter context assembled per visual cell; row context and iterators; CALCULATE as the filter-context editor; ALL/REMOVEFILTERS for % of total; FILTER for row-wise conditions; DIVIDE; cross-fact measures through conformed dimensions; control-total validation.

### Interview questions

1. **Measure vs calculated column?**
   *Columns compute once at refresh in row context and are stored per row — use for attributes you slice by (better yet, do them in Power Query). Measures compute at query time in filter context and are never stored — use for anything aggregated. "If it aggregates, it's a measure."*
2. **Explain filter context in one minute.**
   *The set of filters active where a formula is evaluated — visual coordinates, slicers, page/report filters — propagated through relationships one-to-many to the facts. The same measure returns different values per cell because each cell has different filter context; the grand total is just the near-empty context.*
3. **What does CALCULATE actually do?**
   *Evaluates an expression under a modified filter context: its filter arguments override or add to existing filters on those columns before evaluation. It's the only mainstream way to change context — % of total, time intelligence, and most advanced patterns are CALCULATE with specific filter arguments.*
4. **SUM vs SUMX?**
   *SUM aggregates one existing column. SUMX iterates a table in row context evaluating an expression per row, then sums — needed when the addend must be computed (qty × price). SUM is effectively sugar over SUMX on a single column.*
5. **How do you compute % of total that respects a date slicer but ignores region selection?**
   *Numerator: the base measure. Denominator: CALCULATE of it with REMOVEFILTERS limited to the region/store table only, leaving date filters untouched. The scope of what you remove defines the business question.*
6. **Why DIVIDE instead of `/`?**
   *Graceful divide-by-zero: DIVIDE returns BLANK (or a specified alternate) instead of erroring the visual. Sparse business data hits zero denominators constantly.*
7. **How would you validate a new revenue measure before shipping?**
   *Reconcile the unfiltered total and at least one sliced value against an independent computation — source query, warehouse SQL, or a script over the raw files. Then check edge contexts: grand total, blank dimensions (unknown member), single-day filter.*
8. **Trap: "This measure works in the card but shows the same value for every region — DAX bug?"**
   *Almost never DAX — it means region filters aren't reaching the measure's table: missing/inactive relationship, wrong column used, or the measure hard-overrides region filters with ALL. Diagnose the filter path in Model view first.*

### Common traps

- Answering "CALCULATE filters the data" — it *modifies filter context*; sloppy phrasing signals shallow understanding.
- Reaching for `FILTER(ALL(table), ...)` everywhere — often slower and wrong under other slicers; use targeted boolean filters or REMOVEFILTERS on specific columns.

### Key takeaways

1. Context is the language: filter context per cell, row context per iterated row, CALCULATE as the editor between them.
2. Build small, named, documented measures and compose them.
3. Trust nothing you haven't reconciled to a control number.

**Next up:** [Lab_05_Advanced_DAX_and_Time_Intelligence.md](Lab_05_Advanced_DAX_and_Time_Intelligence.md) — YTD, YoY, and target-vs-actual on the July-1 fiscal calendar, plus variables and disconnected tables.
