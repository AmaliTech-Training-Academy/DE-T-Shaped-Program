# SK-06 END-TO-END GUIDED LAB: Executive Sales Dashboard for NorthStar Retail

---

## SCENARIO: THE BUSINESS PROBLEM

You are a **Power BI developer** deployed to **NorthStar Retail**, a mid-market retail chain with 85 stores across 4 regions, 12,000 SKUs, and 5 product categories. Every Monday morning, the CFO receives a 40-page Excel report manually compiled by 3 analysts over the weekend.

The situation is unsustainable on multiple fronts:

- **30 analyst-hours wasted weekly** on mechanical Excel compilation — analysts spend their weekends copy-pasting, not analysing
- **Stale data**: by Monday morning the report covers data through Friday — the CFO is making decisions on 3-day-old numbers
- **No interactivity**: the report shows region totals but clicking into a region to see store-level detail requires a separate email request to an analyst
- **Multiple versions of truth**: one analyst calculates "revenue" as gross, another as net, causing disagreements in leadership meetings
- **No mobile access**: the CFO travels frequently and cannot check numbers on her phone

**Target:** Replace the 40-page Excel report with a single Power BI dashboard that refreshes daily, allows drill-down from region to store to product, enforces consistent KPI definitions, and works on mobile.

---

## WHY THIS SCENARIO?

| Question | Answer |
|---|---|
| **Why a star schema instead of keeping the source OLTP tables?** | The source schema has 12+ tables joined with 8+ relationships. Every visual would require Power BI to evaluate all those joins. A star schema flattens dimensions so each visual executes 1–3 joins maximum — 10x faster query performance. |
| **Why build the date dimension in Power Query rather than importing from SQL?** | The SQL dim_date doesn't have NorthStar's fiscal calendar (July 1 start). Building it in M means the fiscal logic is in the Power BI model, visible and maintainable by the BI team — not buried in a database object owned by a different team. |
| **Why use measures instead of calculated columns for revenue metrics?** | Calculated columns are computed at refresh time and stored in the model (increasing its size). Measures compute dynamically in the current filter context — this is what allows a measure to show $42M at the company level and $8M at the Northeast region level using the same formula. |
| **Why RLS rather than separate reports per region?** | Five separate reports (one per region + one company) means five-times the maintenance work for every change. One report with RLS means one change propagates to all users automatically. |

---

## WHAT YOU WILL BUILD

```
NorthStar Power BI Solution:
│
├── 1. DATA MODEL (Star Schema)
│   ├── dim_date         (fiscal calendar, built in M)
│   ├── dim_store        (store, district, region hierarchy)
│   ├── dim_product      (product, subcategory, category hierarchy)
│   ├── dim_customer     (customer attributes)
│   ├── dim_promotion    (promotion details)
│   ├── fact_sales       (grain: one row per order line item)
│   └── fact_targets     (monthly revenue targets per region)
│
├── 2. POWER QUERY TRANSFORMATIONS
│   ├── dim_date          (M code — fiscal year July 1, holidays)
│   ├── fact_sales        (SQL source with query folding, date keys)
│   └── All dimensions    (SQL source, type-safe, unused columns removed)
│
├── 3. DAX MEASURE LIBRARY (20+ measures)
│   ├── Base:             Net Revenue, Total Cost, Gross Margin, Margin %, Order Count, AOV
│   ├── Time Intelligence: YTD, QTD, PY, YoY Growth %, 3M Moving Average
│   ├── KPI:              Revenue Target, Variance, Variance %, Status (RAG)
│   └── Advanced:         Customer Segment, New vs Repeat, Pareto %, Ship Date Revenue
│
├── 4. REPORT (5 pages)
│   ├── Page 1: Executive Summary (CFO landing page)
│   ├── Page 2: Regional Performance (drill-through from Page 1)
│   ├── Page 3: Product Analytics
│   ├── Page 4: Customer Analytics
│   └── Page 5: Operational KPIs (shipping, returns)
│
└── 5. ROW-LEVEL SECURITY
    ├── Regional Manager role (sees own region only)
    └── Store Manager role (sees own store only)
```

### Time Estimate: 8–10 hours

---

## PART 1: DATA MODEL DESIGN (45 min)

### Step 1.1: Star Schema Architecture

**WHY this schema:** Every DAX measure in the entire report evaluates by traversing relationships from dimension tables into fact_sales. The fewer hops the better. Denormalizing the product hierarchy (category, subcategory, brand all inside dim_product rather than in separate tables) means a product filter requires one hop, not three.

```
                    dim_date
                      │ DateKey (PK)
                      │ ← order_date_key [ACTIVE]
                      │ ← ship_date_key [INACTIVE]
                      ↓
dim_customer ──→ fact_sales ←── dim_product
  CustomerKey     OrderDateKey     ProductKey
                  ShipDateKey      (one active
dim_store ──→    CustomerKey       relationship
  StoreKey        StoreKey         per table)
                  ProductKey
dim_promotion ─→  PromotionKey
  PromotionKey
                 ↑
            fact_targets
            (separate fact — same dim_date, dim_store relationships)
```

**Key design decisions documented:**
- `fact_sales` grain: one row per order line item
- Two date relationships to `dim_date` (order date active, ship date inactive)
- `dim_product` is fully denormalised — subcategory and category columns live in `dim_product`, not in separate tables
- `dim_store` is fully denormalised — district and region live in `dim_store`
- Integer surrogate keys on all dimension tables (not string GUIDs — better VertiPaq compression)

---

## PART 2: POWER QUERY (M) — DATA PREPARATION (60 min)

### Step 2.1: Date Dimension (Complete M Code)

**WHY build in M (not import from SQL or use auto date/time):** NorthStar's fiscal year starts July 1. The SQL `dim_date` table uses a calendar year. Building in M gives the BI team full ownership of the fiscal calendar logic without a database dependency.

```m
// ═══════════════════════════════════════════════════════════════
// dim_date — Complete Fiscal Date Dimension
// Fiscal Year: July 1 (FY2026 = July 1 2025 – June 30 2026)
// ═══════════════════════════════════════════════════════════════
let
    // ── Configuration ─────────────────────────────────────────
    StartDate            = #date(2020, 1, 1),
    EndDate              = #date(2027, 12, 31),
    FiscalYearStartMonth = 7,   // July

    // ── Generate date list ─────────────────────────────────────
    DayCount   = Duration.Days(EndDate - StartDate) + 1,
    DateList   = List.Dates(StartDate, DayCount, #duration(1, 0, 0, 0)),
    DateTable  = Table.FromList(DateList, Splitter.SplitByNothing(), {"Date"}),
    TypedDate  = Table.TransformColumnTypes(DateTable, {{"Date", type date}}),

    // ── Calendar attributes ────────────────────────────────────
    AddYear          = Table.AddColumn(TypedDate, "Year", each Date.Year([Date]), Int64.Type),
    AddMonthNum      = Table.AddColumn(AddYear, "Month Number", each Date.Month([Date]), Int64.Type),
    AddMonthName     = Table.AddColumn(AddMonthNum, "Month Name", each Date.MonthName([Date]), type text),
    AddMonthShort    = Table.AddColumn(AddMonthName, "Month Short",
                           each Text.Start(Date.MonthName([Date]), 3), type text),
    AddQuarter       = Table.AddColumn(AddMonthShort, "Quarter",
                           each "Q" & Text.From(Date.QuarterOfYear([Date])), type text),
    AddDayOfWeek     = Table.AddColumn(AddQuarter, "Day of Week",
                           each Date.DayOfWeek([Date], Day.Monday) + 1, Int64.Type),
    AddDayName       = Table.AddColumn(AddDayOfWeek, "Day Name",
                           each Date.DayOfWeekName([Date]), type text),
    AddWeekNum       = Table.AddColumn(AddDayName, "Week Number",
                           each Date.WeekOfYear([Date]), Int64.Type),
    AddIsWeekend     = Table.AddColumn(AddWeekNum, "Is Weekend",
                           each Date.DayOfWeek([Date], Day.Monday) >= 5, type logical),

    // ── Fiscal Year (starts July 1) ────────────────────────────
    // FY2026 = July 1 2025 to June 30 2026
    AddFiscalYear    = Table.AddColumn(AddIsWeekend, "Fiscal Year",
                           each if Date.Month([Date]) >= FiscalYearStartMonth
                                then Date.Year([Date]) + 1
                                else Date.Year([Date]),
                           Int64.Type),

    // Fiscal month number: July=1, August=2, ... June=12
    AddFiscalMonth   = Table.AddColumn(AddFiscalYear, "Fiscal Month Number",
                           each Number.Mod(Date.Month([Date]) - FiscalYearStartMonth + 12, 12) + 1,
                           Int64.Type),

    // Fiscal quarter
    AddFiscalQtr     = Table.AddColumn(AddFiscalMonth, "Fiscal Quarter",
                           each "FQ" & Text.From(Number.RoundUp([Fiscal Month Number] / 3)),
                           type text),

    // ── Sort keys ──────────────────────────────────────────────
    // Year-Month for sorting (prevents alphabetic month sort)
    AddYearMonthSort = Table.AddColumn(AddFiscalQtr, "Year-Month Sort",
                           each Date.Year([Date]) * 100 + Date.Month([Date]),
                           Int64.Type),
    AddYearMonth     = Table.AddColumn(AddYearMonthSort, "Year-Month",
                           each Text.From(Date.Year([Date])) & "-" &
                                Text.PadStart(Text.From(Date.Month([Date])), 2, "0"),
                           type text),

    // ── Integer date key for relationships ────────────────────
    // YYYYMMDD format — sorts correctly as an integer
    AddDateKey       = Table.AddColumn(AddYearMonth, "DateKey",
                           each Date.Year([Date]) * 10000
                                + Date.Month([Date]) * 100
                                + Date.Day([Date]),
                           Int64.Type),

    // ── US Federal Holidays (static list) ─────────────────────
    Holidays = #table(
        type table [HolidayDate = date, HolidayName = text],
        {
            {#date(2024, 1, 1),  "New Year's Day"},
            {#date(2024, 7, 4),  "Independence Day"},
            {#date(2024, 11, 28),"Thanksgiving Day"},
            {#date(2024, 12, 25),"Christmas Day"},
            {#date(2025, 1, 1),  "New Year's Day"},
            {#date(2025, 7, 4),  "Independence Day"},
            {#date(2025, 11, 27),"Thanksgiving Day"},
            {#date(2025, 12, 25),"Christmas Day"},
            {#date(2026, 1, 1),  "New Year's Day"},
            {#date(2026, 7, 4),  "Independence Day"}
        }
    ),

    MergeHolidays    = Table.NestedJoin(AddDateKey, {"Date"}, Holidays, {"HolidayDate"},
                           "HolidayMatch", JoinKind.LeftOuter),
    ExpandHolidays   = Table.ExpandTableColumn(MergeHolidays, "HolidayMatch",
                           {"HolidayName"}, {"Holiday Name"}),
    AddIsHoliday     = Table.AddColumn(ExpandHolidays, "Is Holiday",
                           each [Holiday Name] <> null, type logical),

    // ── Final output — reorder and set types ──────────────────
    FinalTable       = Table.SelectColumns(AddIsHoliday, {
                           "DateKey", "Date", "Year", "Month Number", "Month Name",
                           "Month Short", "Quarter", "Day of Week", "Day Name",
                           "Week Number", "Is Weekend", "Is Holiday", "Holiday Name",
                           "Fiscal Year", "Fiscal Month Number", "Fiscal Quarter",
                           "Year-Month", "Year-Month Sort"
                       })

in
    FinalTable
```

### Step 2.2: fact_sales — SQL Source with Query Folding

**WHY use a native SQL query as the source:** Defining the SQL in the Source step ensures ALL transformations (the JOIN of orders + order_lines + products) happen on the SQL Server, not in Power BI memory. If Power BI had imported the three tables and joined them in M, query folding would still work — but the intermediate tables would be unnecessarily large.

```m
// ═══════════════════════════════════════════════════════════════
// fact_sales — Sales Fact Table
// All logic pushed to SQL Server via query folding
// ═══════════════════════════════════════════════════════════════
let
    Source = Sql.Database(
        "northstar-sql.database.windows.net",
        "NorthStarOLTP",
        [
            Query = "
                SELECT
                    o.order_id,
                    o.order_date,
                    o.ship_date,
                    o.customer_id,
                    ol.product_id,
                    ol.store_id,
                    ol.promotion_id,
                    ol.quantity,
                    ol.unit_price,
                    ol.discount_amount,
                    ol.quantity * ol.unit_price                         AS gross_revenue,
                    ol.quantity * ol.unit_price - ol.discount_amount    AS net_revenue,
                    p.unit_cost * ol.quantity                           AS total_cost,
                    CASE WHEN o.is_returned = 1 THEN 1 ELSE 0 END      AS is_returned
                FROM   sales.orders o
                JOIN   sales.order_lines ol ON o.order_id = ol.order_id
                JOIN   products.products p  ON ol.product_id = p.product_id
                WHERE  o.order_date >= '2021-01-01'
            "
        ]
    ),

    // Date keys for the two relationships to dim_date
    AddOrderDateKey = Table.AddColumn(Source, "OrderDateKey",
                          each Date.Year([order_date]) * 10000
                               + Date.Month([order_date]) * 100
                               + Date.Day([order_date]),
                          Int64.Type),

    AddShipDateKey  = Table.AddColumn(AddOrderDateKey, "ShipDateKey",
                          each if [ship_date] = null then null
                               else Date.Year([ship_date]) * 10000
                                    + Date.Month([ship_date]) * 100
                                    + Date.Day([ship_date]),
                          Int64.Type),

    // Gross margin calculated in Power Query (not a DAX calculated column —
    // this is a fixed per-row value, not context-sensitive)
    AddMargin       = Table.AddColumn(AddShipDateKey, "gross_margin",
                          each [net_revenue] - [total_cost],
                          Currency.Type),

    // Remove the raw date columns (relationships use date key integers)
    RemoveDates     = Table.RemoveColumns(AddMargin, {"order_date", "ship_date"}),

    // Explicit types — never let Power BI infer
    TypedTable      = Table.TransformColumnTypes(RemoveDates, {
                          {"order_id",        Int64.Type},
                          {"customer_id",     Int64.Type},
                          {"product_id",      Int64.Type},
                          {"store_id",        Int64.Type},
                          {"quantity",        Int64.Type},
                          {"unit_price",      Currency.Type},
                          {"discount_amount", Currency.Type},
                          {"gross_revenue",   Currency.Type},
                          {"net_revenue",     Currency.Type},
                          {"total_cost",      Currency.Type},
                          {"is_returned",     Int64.Type}
                      })

in
    TypedTable
```

---

## PART 3: DAX MEASURE LIBRARY (90 min)

### Step 3.1: Base Measures

```dax
// ── Naming convention: [Category] Measure Name ─────────────────
// This creates a logical grouping visible in the Fields pane.

// Primary revenue metric — CFO defines revenue as net of discounts
[Sales] Net Revenue =
SUM(fact_sales[net_revenue])

[Sales] Total Cost =
SUM(fact_sales[total_cost])

[Sales] Gross Margin =
[Sales] Net Revenue - [Sales] Total Cost

// WHY DIVIDE not "/": DIVIDE handles division by zero gracefully.
// BLANK() is the correct alternate result — not 0 (which would
// incorrectly suggest a 0% margin rather than no data).
[Sales] Gross Margin % =
DIVIDE([Sales] Gross Margin, [Sales] Net Revenue, BLANK())

[Sales] Order Count =
DISTINCTCOUNT(fact_sales[order_id])

[Sales] Average Order Value =
DIVIDE([Sales] Net Revenue, [Sales] Order Count, BLANK())

[Sales] Quantity Sold =
SUM(fact_sales[quantity])

[Sales] Return Rate =
VAR TotalOrders   = DISTINCTCOUNT(fact_sales[order_id])
VAR ReturnedOrders =
    CALCULATE(
        DISTINCTCOUNT(fact_sales[order_id]),
        fact_sales[is_returned] = 1
    )
RETURN DIVIDE(ReturnedOrders, TotalOrders, BLANK())
```

### Step 3.2: Time Intelligence Measures

```dax
// ── WHY the "6/30" fiscal year end parameter ───────────────────
// NorthStar's fiscal year ends June 30. Without this parameter,
// DATESYTD starts accumulating from January 1. With it, the YTD
// resets at the correct fiscal year boundary.

[TI] Revenue YTD =
CALCULATE(
    [Sales] Net Revenue,
    DATESYTD(dim_date[Date], "6/30")
)

[TI] Revenue QTD =
CALCULATE(
    [Sales] Net Revenue,
    DATESQTD(dim_date[Date])
)

// Prior year same period — the CFO's most-requested benchmark
[TI] Revenue PY =
CALCULATE(
    [Sales] Net Revenue,
    SAMEPERIODLASTYEAR(dim_date[Date])
)

// WHY BLANK() not 0: If there is no prior year data (first year),
// showing 0% growth is misleading. BLANK() correctly shows no comparison.
[TI] YoY Growth % =
VAR CurrentRevenue     = [Sales] Net Revenue
VAR PriorYearRevenue   = [TI] Revenue PY
RETURN
DIVIDE(
    CurrentRevenue - PriorYearRevenue,
    PriorYearRevenue,
    BLANK()
)

// YTD comparison to prior year (for YTD growth tracking)
[TI] Revenue PY YTD =
CALCULATE(
    [Sales] Net Revenue,
    DATESYTD(SAMEPERIODLASTYEAR(dim_date[Date]), "6/30")
)

[TI] YTD Growth % =
DIVIDE(
    [TI] Revenue YTD - [TI] Revenue PY YTD,
    [TI] Revenue PY YTD,
    BLANK()
)

// 3-month moving average smooths holiday/promotion spikes
// Shows the underlying trend, not the noisy month-to-month
[TI] Revenue 3M Moving Avg =
AVERAGEX(
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -3, MONTH),
    CALCULATE([Sales] Net Revenue)
)

// Rolling 12 months (does not reset at year end — unlike YTD)
[TI] Revenue R12M =
CALCULATE(
    [Sales] Net Revenue,
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -12, MONTH)
)
```

### Step 3.3: KPI Measures

```dax
// Revenue Target — from fact_targets table
// WHY USERELATIONSHIP: fact_targets has its own date key column
// (target_date_key) linked to dim_date via an INACTIVE relationship.
// USERELATIONSHIP activates it within this specific measure,
// so the year/month slicer that filters fact_sales also correctly
// filters fact_targets.
[KPI] Revenue Target =
CALCULATE(
    SUM(fact_targets[revenue_target]),
    USERELATIONSHIP(fact_targets[target_date_key], dim_date[DateKey])
)

[KPI] Revenue Variance =
[Sales] Net Revenue - [KPI] Revenue Target

[KPI] Revenue Variance % =
DIVIDE([KPI] Revenue Variance, [KPI] Revenue Target, BLANK())

// Traffic light for conditional formatting on KPI cards
// Returns 1 (green), 0 (yellow), or -1 (red)
[KPI] Revenue Status =
VAR VarPct = [KPI] Revenue Variance %
RETURN
SWITCH(
    TRUE(),
    VarPct >= 0,     1,    // Green: at or above target
    VarPct >= -0.10, 0,    // Yellow: within 10% below target
    -1                     // Red: more than 10% below target
)
```

### Step 3.4: Advanced Measures

```dax
// ── Ship Date Revenue (inactive relationship activation) ────────
// The logistics team reports by ship date, not order date.
// This measure uses USERELATIONSHIP to activate the inactive
// ship_date → dim_date relationship for this measure only.
[Shipping] Revenue by Ship Date =
CALCULATE(
    [Sales] Net Revenue,
    USERELATIONSHIP(fact_sales[ShipDateKey], dim_date[DateKey])
)

[Shipping] Avg Days to Ship =
AVERAGEX(
    fact_sales,
    // Day difference between ship date key and order date key
    // Note: date keys are YYYYMMDD integers — convert to dates first
    DATEDIFF(
        DATE(
            INT(fact_sales[OrderDateKey] / 10000),
            INT(MOD(fact_sales[OrderDateKey], 10000) / 100),
            MOD(fact_sales[OrderDateKey], 100)
        ),
        DATE(
            INT(fact_sales[ShipDateKey] / 10000),
            INT(MOD(fact_sales[ShipDateKey], 10000) / 100),
            MOD(fact_sales[ShipDateKey], 100)
        ),
        DAY
    )
)

// ── Revenue % of Company Total ──────────────────────────────────
// WHY REMOVEFILTERS not ALL:
// ALL(dim_store) would also remove the year slicer's effect if dim_store
// had a date column. REMOVEFILTERS is more precise — it removes only
// the specified columns' filters.
[Analysis] Revenue % of Company Total =
DIVIDE(
    [Sales] Net Revenue,
    CALCULATE(
        [Sales] Net Revenue,
        REMOVEFILTERS(dim_store[region_name]),
        REMOVEFILTERS(dim_store[store_name])
        -- Year slicer (from dim_date) is PRESERVED
    ),
    BLANK()
)

// ── New vs Repeat Customer Revenue ─────────────────────────────
[Advanced] New Customer Revenue =
CALCULATE(
    [Sales] Net Revenue,
    FILTER(
        dim_customer,
        dim_customer[registration_date] >= MIN(dim_date[Date]) &&
        dim_customer[registration_date] <= MAX(dim_date[Date])
    )
)

[Advanced] Repeat Customer Revenue =
[Sales] Net Revenue - [Advanced] New Customer Revenue

[Advanced] Repeat Customer % =
DIVIDE([Advanced] Repeat Customer Revenue, [Sales] Net Revenue, BLANK())
```

---

## PART 4: ROW-LEVEL SECURITY (30 min)

### Step 4.1: Security Mapping Table

First, create a security mapping table in Power Query (or import from SQL):

```m
// ═══════════════════════════════════════════════════════════════
// security_user_region — Maps user emails to their region
// In production: load from a managed SQL table; update via IT
// ═══════════════════════════════════════════════════════════════
let
    SecurityTable = Table.FromRecords({
        [user_email = "ne.manager@northstar.com",   region_name = "Northeast"],
        [user_email = "se.manager@northstar.com",   region_name = "Southeast"],
        [user_email = "mw.manager@northstar.com",   region_name = "Midwest"],
        [user_email = "west.manager@northstar.com", region_name = "West"],
        [user_email = "cfo@northstar.com",          region_name = "__ALL__"]
    }),
    TypedTable = Table.TransformColumnTypes(SecurityTable, {
        {"user_email",  type text},
        {"region_name", type text}
    })
in
    TypedTable
```

### Step 4.2: RLS Role Configuration

**In Power BI Desktop → Modeling → Manage Roles:**

**Role: "Regional Access"** — applied to `dim_store` table:
```dax
// Dynamic RLS: the logged-in user's email determines which region they see.
// USERPRINCIPALNAME() returns the Azure AD email of the current user.
// LOOKUPVALUE looks up their authorized region from the mapping table.
// The filter on dim_store automatically cascades to fact_sales
// through the existing active relationship.

[region_name] = LOOKUPVALUE(
    security_user_region[region_name],
    security_user_region[user_email],
    USERPRINCIPALNAME()
)

// IMPORTANT: This filter returns no rows for users not in the mapping
// table (all data hidden). Always ensure all users are in the mapping
// before publishing. Use an AAD group assignment in Power BI Service
// to assign the "Regional Access" role to all regional managers.
```

**Role: "Store Access"** — applied to `dim_store` table:
```dax
// Even more restrictive — store managers see only their own store
[store_id] = LOOKUPVALUE(
    security_user_store[store_id],
    security_user_store[user_email],
    USERPRINCIPALNAME()
)
```

### Step 4.3: Testing RLS

**In Power BI Desktop:**
1. Modeling tab → View as → Select "Regional Access"
2. Verify the Executive Summary page shows only aggregate data (no region breakdown visible)
3. Click "Other user" → enter `ne.manager@northstar.com` → verify only Northeast data appears
4. Verify all measures (Revenue, Margin, Order Count) show only Northeast values
5. **Critical negative test:** Verify that a Northeast user cannot see Southeast data even by modifying URL parameters

---

## PART 5: REPORT DESIGN (60 min)

### Step 5.1: Page 1 — Executive Summary Layout

```
LAYOUT SPECIFICATION:
─────────────────────────────────────────────────────────────────

ROW 1 — FILTER BAR (top of page, full width)
  [Fiscal Year slicer — dropdown]  [Quarter slicer — buttons]  [Region slicer — buttons]

ROW 2 — KPI CARDS (4 equally-spaced cards)
  Card 1: Net Revenue ($42.3M)
    Subtitle: vs Prior Year ($39.1M)
    Color: Green if YoY > 0, Red if YoY < 0
    Trend axis: Monthly revenue (line sparkline)

  Card 2: YoY Growth (+8.2%)
    Conditional icon: ▲ green / ▼ red
    Reference label: "Target: 5.0%"

  Card 3: Gross Margin % (34.5%)
    Conditional color: Green if > 35%, Yellow if 30-35%, Red if < 30%
    Subtitle: vs PY (35.7%)

  Card 4: Order Count (182K)
    Subtitle: +5.1% vs PY
    Trend axis: Monthly order count

ROW 3 — LEFT CHART: Revenue Trend (60% width)
  Line chart: X axis = Month Name (sorted by Year-Month Sort)
  Lines:
    - Net Revenue CY (solid blue #2196F3)
    - Net Revenue PY (dashed grey #9E9E9E)
    - Revenue 3M Moving Avg (dotted orange #FF9800)
  Target line: Revenue Target (dashed red, from fact_targets)
  Title: "Revenue Trend — Current Year vs Prior Year"

ROW 3 — RIGHT CHART: Revenue by Region (40% width)
  Clustered bar chart (horizontal — regions are long text)
  Bars: Net Revenue CY (blue) + Revenue Target (outlined)
  Conditional bar color: Green if >= target, Red if < target
  Drill-through: Right-click any bar → Drill through → Page 2

ROW 4 — LEFT TABLE: Top 10 Products (60% width)
  Table columns: Product Name, Revenue, % of Total, YoY Growth %, Margin %
  Conditional formatting:
    - YoY Growth %: icon set (▲ green, ► yellow, ▼ red)
    - Margin %: data bars (light blue gradient)
  Sort: by Revenue descending

ROW 4 — RIGHT CHART: Revenue by Category (40% width)
  Donut chart: 5 segments (one per category)
  Legend: visible, positioned right
  Labels: show % of total
```

### Step 5.2: Drill-Through Configuration for Page 2

On **Page 2 (Regional Performance)**:
1. Add "Region Name" from `dim_store` to the **Drill-through** field well
2. This enables right-click → "Drill through" from any visual on Page 1 that has region context
3. Add a Back button (Insert → Buttons → Back) — Power BI auto-configures it to return to the source page
4. Page 2 should show store-level breakdowns, a map visual, and store-vs-target comparison

---

## PART 6: PERFORMANCE OPTIMIZATION (30 min)

### Step 6.1: Performance Analyzer Walkthrough

```
HOW TO USE PERFORMANCE ANALYZER:
1. Power BI Desktop → View → Performance Analyzer
2. Click "Start recording"
3. Interact with the Executive Summary page (change slicer, click a bar)
4. Click "Stop"
5. Expand each visual to see three time components:
   - DAX query: time for the formula engine to calculate measures
   - Direct Query: time waiting for the source database (0 if Import mode)
   - Other: visual rendering time

TARGET:
  < 200ms  = Excellent (don't touch)
  200-500ms = Good (monitor)
  500ms-2s  = Investigate the DAX
  > 2s      = Must optimize before production

WHAT TO DO WITH A SLOW VISUAL:
1. Click "Copy query" next to the slow visual
2. Paste into DAX Studio
3. Run with Server Timings enabled
4. Look for: high FE (formula engine) time = slow DAX
           high SE (storage engine) queries = data volume issue
```

### Step 6.2: Model Optimization Checklist

```
OPTIMIZATION 1: Remove Unused Columns ─────────────────────────
Impact: Largest single optimization — reduces model size 30-70%
Action: In Power Query, remove all columns not referenced in:
  • Any measure
  • Any calculated column
  • Any visual field well
  • Any relationship
  
Columns typically removed from fact_sales:
  - created_by, modified_by (audit fields)
  - internal_notes, legacy_code (system fields)
  - batch_number, source_system, etl_timestamp (pipeline fields)
  - payment_gateway_response, ip_address, session_id (web fields)

OPTIMIZATION 2: Integer Keys ───────────────────────────────────
Impact: Reduces relationship column size by ~80%
BEFORE: customer_id = "CUST-NE-00042837" (VARCHAR, ~20 bytes)
AFTER:  customer_key = 42837 (INT, 4 bytes)
Action: In Power Query, create integer surrogate keys

OPTIMIZATION 3: Measures over Calculated Columns ───────────────
Impact: Reduces model size; measures compute only when needed
Rule: If the value changes based on slicer selection → MUST be measure
If the value is a fixed row attribute → consider calculated column
      (but Power Query is usually better)
BEFORE (calculated column — stored per row):
  margin_pct = DIVIDE(fact_sales[net_revenue] - fact_sales[total_cost],
                      fact_sales[net_revenue])
AFTER (measure — no storage cost):
  [Sales] Gross Margin % = DIVIDE([Sales] Gross Margin, [Sales] Net Revenue, BLANK())

OPTIMIZATION 4: Variables in DAX ──────────────────────────────
Impact: Prevents redundant storage engine queries
Rule: If a sub-expression is referenced more than once → use VAR
BEFORE (calculates [TI] Revenue PY twice):
  YoY % = DIVIDE([Sales] Net Revenue - [TI] Revenue PY, [TI] Revenue PY)
AFTER (calculates [TI] Revenue PY once):
  YoY % = VAR CurrentRev = [Sales] Net Revenue
           VAR PriorRev   = [TI] Revenue PY
           RETURN DIVIDE(CurrentRev - PriorRev, PriorRev, BLANK())

OPTIMIZATION 5: Disable Auto Date/Time ─────────────────────────
Impact: Removes hidden date tables (one per date column in the model)
Action: File → Options → Data Load → Uncheck "Auto date/time"
        (Then mark your custom dim_date as the date table)
```

---

## PART 7: DEPLOYMENT (20 min)

### Step 7.1: Publish to Power BI Service

```
DEPLOYMENT PIPELINE SETUP:

[Dev Workspace]    [Test Workspace]   [Production Workspace]
  ↑                    ↑                    ↑
Developers publish   QA validates data    End users consume
No restrictions      and performance      Viewer access only
                                          2 people can deploy

STEPS:
1. Create 3 workspaces: NorthStar-Dev, NorthStar-Test, NorthStar-Prod
2. Power BI Service → Deployment Pipelines → Create a pipeline
3. Assign workspaces: Dev → Test → Prod
4. Add deployment rules:
   - Dev:  connects to northstar-dev.database.windows.net
   - Test: connects to northstar-test.database.windows.net
   - Prod: connects to northstar-sql.database.windows.net

PUBLISH WORKFLOW:
Developer → Publish to Dev workspace → Deploy to Test →
QA validates → Team lead approves → Deploy to Production

REFRESH SCHEDULE (Production):
- Daily at 5:00 AM (before business hours)
- Failure notification: data-team@northstar.com
- Timeout: 2 hours
- Gateway: NorthStar-PBI-Gateway (on-prem SQL connection)
```

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The `fact_targets` table links to `dim_date` via an inactive relationship. The CFO asks you to create a Revenue vs Target visual that responds to the Year and Quarter slicers. You use USERELATIONSHIP in the Revenue Target measure. But when you add both Revenue and Revenue Target to the same visual, Revenue responds to the slicer but Revenue Target always shows the full year total. What is most likely wrong?

2. The date dimension M code uses `Date.DayOfWeek([Date], Day.Monday)` to compute the day of week. What is the difference between `Day.Monday` and `Day.Sunday` as the second parameter, and which one is correct for a US retailer reporting dashboard?

3. A regional manager complains: "When I open the report, I see all regions' data, not just mine." You know RLS is configured correctly because "View as" in Desktop shows the correct filtering. What are the 3 most likely causes of this issue in the deployed Power BI Service environment?

4. The Performance Analyzer shows that the "Revenue Trend" line chart takes 4.2 seconds: DAX query = 3.8s, Other = 0.4s. You open the DAX in DAX Studio with Server Timings. You see 12 storage engine (SE) queries and 1 formula engine (FE) pass. What does the high SE query count suggest about the DAX measure, and what pattern would you change to reduce it?

5. The CFO asks you to add a "Budget Variance by Product Category" visual, but the `fact_targets` table only has targets at the region level — not the product category level. Describe two approaches to handle this (one using the existing data model, one requiring a data model change), and explain the trade-off.
