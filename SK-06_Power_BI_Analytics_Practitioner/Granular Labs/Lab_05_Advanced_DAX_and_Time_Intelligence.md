# Lab 05 — Advanced DAX & Time Intelligence

> **Module:** SK-06 Power BI Analytics Practitioner
> **Estimated time:** 7–8 hours
> **Builds on:** Lab_04 — open `NorthStar_Lab04.pbix`, **Save As** `NorthStar_Lab05.pbix`.
> **Produces:** `NorthStar_Lab05.pbix` — a ~20-measure library including fiscal YTD, YoY, moving averages, a what-if parameter, and a semi-additive preview for the capstone.

"How are we doing **versus last year**?" is the most-asked question in business analytics. This lab makes you fluent in time intelligence — on a **July-1 fiscal year**, which is where copy-paste DAX from blogs quietly breaks — plus the patterns that separate practitioners from beginners: variables, inactive relationships, disconnected tables, and semi-additive measures.

---

## 1. Environment Setup

1. Open `NorthStar_Lab04.pbix` → Save As `C:\PowerBI_Labs\NorthStar\pbix\NorthStar_Lab05.pbix`.
2. Non-negotiable preconditions (each breaks this lab if missing):
   - `dim_date` exists, is **marked as date table**, spans the full data range (Lab_03 Step 4–5);
   - fiscal columns verified: 2025-07-01 → FY2026/Q1 (re-check in Table view — 30 seconds now saves an hour later);
   - Auto date/time OFF.
3. Optional: DAX Studio (Lab_04) for timing comparisons in Step 6.

---

## 2. Business Context

NorthStar's CFO doesn't ask "what was revenue?" — she asks "what was revenue *versus last year*, *versus target*, and *are we on pace* for the fiscal year?" Absolute numbers are meaningless without comparison baselines; time is the universal baseline.

Two business realities make this technically non-trivial:

1. **The fiscal calendar.** NorthStar's year runs July 1 – June 30 (retail planning cycles often do; governments and universities too). "YTD" in October means *July–October*, not January–October. Every out-of-the-box demo assumes calendar years; real engagements rarely do. The capstone uses the same fiscal convention (C-03) — what you build here transfers directly.
2. **Not everything sums over time.** Revenue does. Account balances, inventory levels, and **headcount** don't — Q1 headcount is the *end-of-quarter* value, not the sum of three months. These are **semi-additive** measures, the make-or-break concept of the CloudHR capstone (FR-10), previewed here on NorthStar data.

**If time intelligence is wrong**, the failure is vicious: numbers look plausible (a YoY of +8.3% looks exactly as credible as the correct +5.1%), get quoted in board decks, and steer decisions. Nobody catches it until someone reconciles by hand.

---

## 3. Concept Explanation

### 3.1 How time intelligence works under the hood

Every time-intelligence function is **CALCULATE with a date-table filter**. `DATESYTD(dim_date[Date], "6-30")` returns the set of dates from the fiscal year start through the last date in filter context; CALCULATE swaps that set in as the date filter. That's it. Understanding this de-mystifies the whole function family — and explains the prerequisites: a proper, marked, contiguous date table, and date fields in visuals taken **from dim_date**.

Function families you'll use:

- **To-date:** `DATESYTD` (takes a fiscal year-end parameter — our lever), `TOTALYTD` (sugar for CALCULATE+DATESYTD).
- **Shifts:** `SAMEPERIODLASTYEAR`, `DATEADD(dates, -1, YEAR)` (more general: months/quarters too).
- **Windows:** `DATESINPERIOD` (rolling N months — moving averages).
- **Boundaries:** `LASTDATE`, `ENDOFMONTH` — the backbone of semi-additive patterns.

### 3.2 Variables (VAR/RETURN)

```dax
VAR CurrentValue = [Net Revenue]
VAR PriorValue   = [Net Revenue PY]
RETURN DIVIDE ( CurrentValue - PriorValue, PriorValue )
```

Variables are evaluated **once, where declared**, then reused as constants — better performance (no recompute), better readability, and crucially a *frozen* value you can carry into a modified context. All non-trivial measures from here on use them.

### 3.3 Inactive relationships & USERELATIONSHIP

Lab_03 allowed only one active relationship between two tables. When a fact has two dates (ordered vs shipped), the second relationship exists but sleeps, and a measure wakes it: `CALCULATE([x], USERELATIONSHIP(dim_date[Date], fact[ship_date]))`. Pattern shown in Step 5; the capstone's leave fact uses it.

### 3.4 Disconnected tables & what-if parameters

A table with **no relationships** can't filter anything — which is the point: users pick a value from it (via slicer), and measures read the selection with `SELECTEDVALUE` and *act* on it in logic. This powers what-if analysis ("what if we raised prices 5%?") and measure-selector patterns. Power BI's **New parameter** feature generates the table + slicer + measure for you.

### 3.5 Additivity (the capstone's heart, previewed)

- **Additive:** sums across every dimension (revenue).
- **Semi-additive:** sums across some dimensions but not time (balances, inventory, headcount) — over time you take the *last* (or first) value instead.
- **Non-additive:** never sums (ratios, percentages) — recompute from components at each level, never average the averages.

---

## 4. Step-by-Step Implementation

### Step 1 — Fiscal YTD

Right-click `_Measures` → New measure:

```dax
Net Revenue FYTD =
CALCULATE (
    [Net Revenue],
    DATESYTD ( dim_date[Date], "6-30" )
)
```

The `"6-30"` argument is the fiscal year **end**; omit it and you get calendar YTD — the single most common fiscal-DAX bug.

**Verification (do it properly):** matrix — rows `dim_date[Fiscal Year]` then `dim_date[Month]` (drill both levels on), values `Net Revenue` and `Net Revenue FYTD`.
- July of each fiscal year: FYTD **equals** the monthly value (year just restarted). This is the smoking-gun check: if July shows a large accumulated value instead, your `"6-30"` is missing or wrong.
- Each subsequent month: FYTD = previous FYTD + month value (spot-check one addition by hand).

**Troubleshooting:** FYTD blank everywhere → date field in the visual isn't from `dim_date`, or the table isn't marked as date table. FYTD equals Net Revenue everywhere → the relationship dim_date→fact_sales is missing/inactive.

### Step 2 — Prior year and YoY

```dax
Net Revenue PY =
CALCULATE ( [Net Revenue], SAMEPERIODLASTYEAR ( dim_date[Date] ) )
```

```dax
Net Revenue YoY % =
VAR Current = [Net Revenue]
VAR Prior   = [Net Revenue PY]
RETURN
    IF ( NOT ISBLANK ( Prior ), DIVIDE ( Current - Prior, Prior ) )
```

Why the `ISBLANK` guard: the first year of data has no prior year; without the guard some visuals show misleading blanks-as-zero or infinite-looking growth. Explicitly returning blank keeps charts honest. Format as percentage, 1 decimal.

**Verification:** matrix rows `Month`, values Net Revenue / PY / YoY %. Pick one month; confirm PY equals the same month one year earlier by eye. FY1 months show blank YoY — correct, not a bug.

Note `SAMEPERIODLASTYEAR` needs no fiscal parameter — it shifts the *currently filtered* dates back a year, and your fiscal columns did the grouping. Fiscal awareness lives in `dim_date` + `DATESYTD`; shifts are calendar-agnostic.

Also add the composite the CFO actually reads:

```dax
Net Revenue FYTD PY =
CALCULATE ( [Net Revenue FYTD], SAMEPERIODLASTYEAR ( dim_date[Date] ) )
```

```dax
Net Revenue FYTD YoY % =
VAR C = [Net Revenue FYTD]
VAR P = [Net Revenue FYTD PY]
RETURN IF ( NOT ISBLANK ( P ), DIVIDE ( C - P, P ) )
```

Composition of measures (Lab_04's practice) means each layer was already verified.

### Step 3 — Rolling 3-month average

Monthly retail data is noisy; smoothing reveals trend:

```dax
Net Revenue 3MMA =
VAR Window =
    DATESINPERIOD ( dim_date[Date], MAX ( dim_date[Date] ), -3, MONTH )
RETURN
    DIVIDE ( CALCULATE ( [Net Revenue], Window ), 3 )
```

`MAX(dim_date[Date])` = last date in the current cell's context (that month's end); `DATESINPERIOD` reaches back 3 months from there. **Verification:** line chart, `Month` on X, `Net Revenue` and `Net Revenue 3MMA` — the MMA visibly smooths the raw line and lags turning points slightly (expected). **Common mistake:** dividing by 3 when the window's first two months fall before data starts — acceptable for the lab; the robust version divides by the count of months actually present (`COUNTROWS(VALUES(dim_date[Month Sort]))` over the window). Try it as a stretch goal.

### Step 4 — Target attainment, time-intelligent

Combining Lab_04's cross-fact measures with FYTD:

```dax
Target Revenue FYTD =
CALCULATE ( [Target Revenue], DATESYTD ( dim_date[Date], "6-30" ) )
```

```dax
FYTD Attainment % =
DIVIDE ( [Net Revenue FYTD], [Target Revenue FYTD] )
```

**Verification:** matrix — rows `dim_region[region]`, columns `Fiscal Year`, value `FYTD Attainment %`: plausible percentages (roughly 80–120% given Lab_00's generation), varying by region. This measure answers "are we on pace" — arguably the most valuable single number in the model.

### Step 5 — What-if: price-change simulator (disconnected table)

1. **Modeling ribbon → New parameter → Numeric range**: Name `Price Change %`, Min `-0.10`, Max `0.10`, Increment `0.01`, Default `0` → check "Add slicer to this page" → Create. Power BI generates a disconnected table plus a `Price Change % Value` measure (`SELECTEDVALUE`).
2. New measure:

   ```dax
   Simulated Net Revenue =
   VAR Uplift = [Price Change % Value]
   RETURN
   SUMX (
       fact_sales,
       fact_sales[quantity] * fact_sales[unit_price] * ( 1 + Uplift )
           * ( 1 - fact_sales[discount_pct] )
   )
   ```

   (Deliberately naive — it assumes demand doesn't respond to price. Say that limitation out loud; consultants who state model assumptions get rehired.)
3. **Verification:** card with `Simulated Net Revenue` next to `Net Revenue`; slider at 0 → equal; at +5% → simulated is exactly 5% higher. The slicer table filters *nothing* (check Model view: no relationship lines) yet drives the number — that's the disconnected pattern.

### Step 6 — Semi-additive preview: month-end inventory-style measure

NorthStar has no balance fact, so simulate the *pattern* on what you have — "cumulative units sold to date" as a stand-in stock level, then read it semi-additively:

```dax
Cumulative Units =
CALCULATE (
    [Total Quantity],
    FILTER ( ALL ( dim_date[Date] ), dim_date[Date] <= MAX ( dim_date[Date] ) )
)
```

```dax
Cumulative Units EOP =
CALCULATE ( [Cumulative Units], LASTDATE ( dim_date[Date] ) )
```

Matrix: rows `Fiscal Year` → `Month`, values both measures. At month level they agree. **At the year level** `Cumulative Units` shows the value at the year's last date — and `EOP` makes the intent explicit: *end-of-period value, not a sum across months*. That `LASTDATE` move is precisely FR-10 in the capstone (headcount), where getting it wrong is an automatic fail. Sit with this matrix until the "quarter = last month, not sum of months" behavior feels obvious.

**Common mistake:** wrapping LASTDATE around dates that have no fact rows (e.g., future dates in dim_date) returns blank — the capstone version uses `LASTNONBLANK` variants for that; noted here so the term isn't new when you meet it.

### Step 7 — Housekeeping and save

1. Display folders: Model view → select measures → **Properties → Display folder** — group into `Base`, `Time Intelligence`, `Targets`, `What-If`. Twenty measures unorganized is where discoverability starts dying.
2. Descriptions on every new measure (one business sentence each — Lab_04 habit).
3. Re-run a control check: `Net Revenue` grand total unchanged since Lab_04 (nothing you did should alter base numbers — if it did, hunt the change).
4. Save `NorthStar_Lab05.pbix`.

---

## 5. Production Engineering Practices

- **Fiscal logic lives in ONE place** — `dim_date` plus the `"6-30"` parameter — never re-derived per measure. *Failure story:* an org had "FY" logic re-implemented in 14 measures by 3 authors; two used `>= 7`, one used `> 7`. July numbers disagreed between report pages for a year, and every month-end close included an hour of "which page is right?" Centralize or suffer.
- **Guard the edges.** First-year YoY, empty windows, divide-by-zero: every boundary gets an explicit `IF/ISBLANK/DIVIDE` decision. Blank is a *statement* ("no comparable period"), zero is a *value* — never let one impersonate the other.
- **State model assumptions in writing** (the what-if's naive elasticity). Un-stated assumptions become someone else's incorrect decisions.
- **Regression-check on every change:** base totals must not move when you add derived measures. Keep one "control card" page in the file — it's your unit test suite until real testing tools (DAX queries, PBIP+Git) enter in Lab_06.
- **Performance seed:** variables avoid recomputation; targeted filters beat `FILTER(ALL(...))` scans. Lab_06 measures this with Performance Analyzer; write it well now.

---

## 6. Reflection

### What you learned

Time intelligence as CALCULATE+date-filters; fiscal YTD via `DATESYTD(..., "6-30")`; PY/YoY with blank-guarding; rolling windows; cross-fact time-intelligent attainment; variables; disconnected what-if tables; the semi-additive LASTDATE pattern that the capstone lives on.

### Interview questions

1. **What does a time-intelligence function actually do?**
   *It's CALCULATE with a computed date filter: DATESYTD returns year-start-to-current-max dates and swaps them into filter context. That's why it requires a contiguous, marked date table and date fields drawn from it.*
2. **Your company's fiscal year starts in July. What changes in your DAX?**
   *DATESYTD/TOTALYTD get the year-end parameter ("6-30"); the date dimension carries fiscal year/quarter/month columns for grouping and sorting. Pure shifts like SAMEPERIODLASTYEAR don't change — they move the filtered dates regardless of calendar.*
3. **Why did your YoY measure return blank for the first year, and is that correct?**
   *No prior-year data exists, so the comparison is undefined; returning blank (guarded via ISBLANK) is correct and honest. Returning 0 would claim "no growth"; showing an error would break the visual.*
4. **Additive vs semi-additive vs non-additive — one example each and the DAX consequence?**
   *Revenue is additive: SUM everywhere. Headcount/balances are semi-additive: over time take end-of-period (LASTDATE-style CALCULATE), sum across other dimensions. Ratios are non-additive: recompute from components at every level — never average averages.*
5. **What is a disconnected table and when do you use one?**
   *A table with no relationships, used as a user-selection surface: a slicer sets a value, SELECTEDVALUE reads it, measures act on it. Standard for what-if parameters, scenario pickers, and dynamic measure selectors.*
6. **Why use VAR in measures?**
   *Single evaluation (performance), readability, and — critically — variables freeze a value in the context where declared, so you can carry "current" values into CALCULATE-modified contexts without them being re-evaluated there.*
7. **A YTD measure returns the same value as the base measure. Diagnose.**
   *The date filter substitution isn't happening: visual uses a date column not from the marked date table, the date table isn't marked, the relationship to the fact is missing/inactive, or Auto date/time hierarchies are intercepting. Check the filter path before the formula.*
8. **Trap: "Just average the monthly return-rate percentages to get the quarterly rate."**
   *Non-additive trap: an average of ratios ignores differing denominators. Aggregate the numerator and denominator over the quarter and divide — which happens automatically when the ratio is a measure built from component measures.*

### Common traps

- Quoting `TOTALYTD` without knowing it's sugar over `CALCULATE`+`DATESYTD` — interviewers probe one level down.
- Forgetting the fiscal parameter and shipping calendar YTD to a fiscal-year business (plausible numbers, wrong by construction).

### Key takeaways

1. Time intelligence = CALCULATE + a date-table filter. Prerequisites (marked, contiguous, dim_date-sourced fields) aren't bureaucracy; they're the mechanism.
2. One fiscal definition, one place, guarded edges, composed measures.
3. Semi-additivity is a business fact, not a DAX trick — the measure must match how the number behaves in reality. The capstone will test exactly this.

**Next up:** [Lab_06_Report_Design_Dashboards_RLS_Performance_Publishing.md](Lab_06_Report_Design_Dashboards_RLS_Performance_Publishing.md) — turn the model into a report people use, secure it with RLS, tune it, and learn how it ships.
