# Lab 06 — Report Design, Dashboards, RLS, Performance & Publishing

> **Goal:** Turn the model into the CFO's report: the Executive Summary page and the drill-through Regional Performance page from the guided lab, dynamic row-level security for regional and store managers (tested, including the "nobody" user), a measured performance pass, a mobile layout and a publishing plan.
> **Time:** 6–7 hours | **Difficulty:** Intermediate | **Prerequisites:** Lab 05 (`NorthStar_Lab05.pbix` with the 27-measure library)
> **Guided lab coverage:** PART 4 (row-level security), PART 5 (report design), PART 6 (performance) and PART 7 (deployment)
> **Produces:** `C:\DataEngineering\sk06\pbix\NorthStar_Lab06.pbix`

## Learning objectives

By the end of this lab you can:

- Design a page that passes the **5-second test**: one question per page, the answer top-left, colour reserved for the message.
- Build KPI cards, a trend chart, a target comparison with conditional colours, a Top-N table and drill-through.
- Implement **dynamic RLS** with `USERPRINCIPALNAME()` and mapping tables — including an "all regions" user — and test it from the outside.
- Measure visuals with **Performance Analyzer** and apply the guided lab's optimisation checklist.
- Explain workspaces, apps, gateways, scheduled refresh and deployment pipelines, and what each needs (licence, capacity, gateway).

**Key terms**

| Term | Meaning |
|---|---|
| **Drill-through** | Right-click a data point → jump to a detail page filtered to that point. |
| **Conditional formatting** | Colours/icons driven by a measure (here `Revenue Status` and `YoY Growth %`). |
| **RLS (row-level security)** | A role = a DAX filter on a table. Users in the role only see rows that pass it — in every visual, every report on the model. |
| **USERPRINCIPALNAME()** | The signed-in user's name (UPN, usually their email) in the Service. |
| **Performance Analyzer** | Desktop pane that times every visual: DAX query, Direct query, Other. |
| **Workspace / App** | Team container for content in the Service / the packaged read-only distribution to consumers. |
| **Gateway** | Software that lets the Service refresh data from sources on a private network or a laptop (like your container). |
| **Deployment pipeline** | Dev → Test → Prod promotion of content, with rules that swap parameters per stage. Needs Premium Per User, Premium or Fabric capacity. |

### Design rules you will apply

Readers scan a page like a **Z**: top-left → top-right → down. So: filters in a thin top strip, 3–5 KPI cards next, one main "why" visual, detail at the bottom. Trend → line chart; comparison across categories → bar chart (horizontal for long labels); part-to-whole with few parts → donut; a number vs target → card with a coloured status. Strong colour only for the message (current year, red/green status); grey for references (prior year).

## Steps

### Step 1 — Open a copy and wireframe

1. `docker compose start` in `C:\DataEngineering\sk06`.
2. Open `NorthStar_Lab05.pbix` → **Save As** `C:\DataEngineering\sk06\pbix\NorthStar_Lab06.pbix`. Delete the **TI Test** page from Lab 05 (or keep it hidden: right-click the tab → *Hide page*).
3. On paper, sketch the guided lab's Executive Summary (Part 5.1) — four rows:

```
ROW 1  [Fiscal Year ▼] [Fiscal Quarter: FQ1 FQ2 FQ3 FQ4] [Region: NE SE MW W]
ROW 2  [Net Revenue] [YoY Growth] [Gross Margin %] [Order Count]
ROW 3  [Revenue Trend — line, 60%]                  [Revenue by Region — bars, 40%]
ROW 4  [Top 10 Products — table, 60%]               [Revenue by Category — donut, 40%]
```

Write the page's question above it: *"How is NorthStar doing this fiscal year, and where?"*

✅ **Checkpoint:** a wireframe with a one-sentence question.

### Step 2 — Row 1: the filter bar

1. Rename page 1 to **Executive Summary**.
2. **Slicer** with `dim_date[Fiscal Year]` → Format → Slicer settings → Style **Dropdown**, single select. In the **Filters** pane for this visual, keep only 2024, 2025, 2026 (`dim_date` covers 2020–2028, but only these years have sales). Select **2026**.
3. **Slicer** with `dim_date[Fiscal Quarter]` → Style **Tile** (buttons), multi-select allowed.
4. **Slicer** with `dim_store[region_name]` → Style **Tile**.
5. Align the three across the top (**Format → Align → Align top**, **Distribute horizontally**).

✅ **Checkpoint:** three slicers in one strip; Fiscal Year = 2026.

### Step 3 — Row 2: KPI cards

Insert four **Card** visuals (use the newer *Card* with reference labels if your version has it; otherwise a classic card plus a small text card underneath):

| Card | Value | Reference label / subtitle | Expected (FY2026, no other filter) |
|---|---|---|---|
| 1 | `Net Revenue` | `Revenue PY` ("vs Prior Year") | **$11.86M**, PY $10.78M |
| 2 | `YoY Growth %` | text "Target: 5.0%" | **10.01%** |
| 3 | `Gross Margin %` | PY margin (create `Gross Margin % PY = CALCULATE([Gross Margin %], SAMEPERIODLASTYEAR(dim_date[Date]))` in folder `Time Intelligence`) | **35.47%**, PY 35.49% |
| 4 | `Order Count` | `Order Count` growth vs PY | **63.88K**, +8.74% vs PY (58,745) |

Conditional colour (select the card → Format → Callout value → Color → **fx**):

- Card 1 and 2: **Format style: Rules**, based on field `YoY Growth %`: `>= 0` green, `< 0` red.
- Card 3: Rules on `Gross Margin %`: `> 0.35` green, `>= 0.30 and <= 0.35` amber, `< 0.30` red → FY2026 shows **green**.

For the Order Count growth label, add a small measure (folder `Time Intelligence`, format Percentage 1 decimal):

```dax
Order Count YoY % =
VAR CurrentOrders = [Order Count]
VAR PriorOrders   = CALCULATE([Order Count], SAMEPERIODLASTYEAR(dim_date[Date]))
RETURN DIVIDE(CurrentOrders - PriorOrders, PriorOrders, BLANK())
```

**Verify:** change Fiscal Year to 2025 — Card 1 shows $10.78M with YoY **8.28%**; FY2024 shows blank growth (no FY2023). Back to 2026.

✅ **Checkpoint:** the four cards match the table; colours change with the year.

### Step 4 — Row 3: trend and region comparison

**Revenue Trend (left, ~60% width):** **Line chart**

- X-axis: `dim_date[Year-Month]` (sorted by `Year-Month Sort` since Lab 03). Not `Month Name`: across several years it repeats, so it cannot carry the year-month order.
- Y-axis: `Net Revenue`, `Revenue PY`, `Revenue 3M Moving Avg`, `Revenue Target`.
- Format → **Lines**: per series set colour and style — Net Revenue solid blue `#2196F3` (width 3), Revenue PY dashed grey `#9E9E9E`, 3M Moving Avg dotted orange `#FF9800`, Target dashed red.
- Title: **"Revenue Trend — Current Year vs Prior Year"**.

**Expected output (FY2026):** 12 points from 2025-07 to 2026-06; the blue line peaks in **2025-12 at $1.35M**; the target line sits just above the blue line in every month (NorthStar is 3.7–4.7% below target each month, 4.21% for the year).

**Revenue by Region (right, ~40%):** **Clustered bar chart** (horizontal)

- Y-axis `dim_store[region_name]`; X-axis `Net Revenue` and `Revenue Target`.
- Bar colours: Format → Bars → `Net Revenue` → Color → **fx** → Format style **Rules**, based on field `Revenue Status`: `= 1` green, `= 0` amber, `= -1` red. Make the target series light grey with a border, so the eye compares each bar with its target.

**Expected output (FY2026):** Northeast **green** ($3.35M vs $3.24M), Southeast and Midwest **amber**, West **red** ($2.36M vs $2.69M, −12.27%).

✅ **Checkpoint:** four coloured regions — 1 green, 2 amber, 1 red.

### Step 5 — Row 4: top products and category mix

**Top 10 Products (left):** **Table** with `dim_product[product_name]`, `Net Revenue`, `Revenue % of Company Total`, `YoY Growth %`, `Gross Margin %`.

- Filters pane → `product_name` → Filter type **Top N** → Show items **Top 10** → By value **Net Revenue** → *Apply filter*.
- Conditional formatting (column dropdown in the Values well → *Conditional formatting*): `YoY Growth %` → **Icons** (▲ green / ► amber / ▼ red); `Gross Margin %` → **Data bars**, light blue.
- Sort by `Net Revenue` descending.

**Expected output (FY2026):** top three — **Northwind Cycling 21007** ($198,522.59), **Apex Computing 13378** ($164,249.78), **Lumen Cycling 21325** ($159,302.30).

**Revenue by Category (right):** **Donut chart**, Legend `dim_product[category_name]`, Values `Net Revenue`, detail labels *Percent of total*, legend on the right.

**Expected output (FY2026):** Sports & Outdoors **27.26%**, Home & Garden 24.81%, Electronics 22.01%, Apparel 15.17%, Grocery 10.76%.

Now apply the **5-second test**: show the page to someone (or squint). Can they say "revenue is up 10% on last year, but 4% below target; the West is the problem" within five seconds? If not, remove or shrink something — the fix is nearly always subtraction.

✅ **Checkpoint:** the page answers its question in five seconds; numbers match.

### Step 6 — Page 2: Regional Performance (drill-through)

1. New page **Regional Performance**.
2. Drag `dim_store[region_name]` into the page's **Drill through** well (Visualizations pane → *Build visual* → Drill through). Power BI adds a **Back** button top-left.
3. Add a card with the selected region's name:

```dax
Selected Region =
SELECTEDVALUE(dim_store[region_name], "All regions")
```

4. Add a **Table**: `dim_store[store_name]`, `Net Revenue`, `Revenue Target`, `Revenue Variance %`, with icons on `Revenue Variance %`.
5. Add a **Map**: Latitude `dim_store[latitude]`, Longitude `dim_store[longitude]`, Bubble size `Net Revenue`, Tooltips `store_name`. If the map stays empty, enable **File → Options → GLOBAL → Security → Use Map and Filled Map visuals**.
6. Go back to **Executive Summary**, right-click the **Northeast** bar → **Drill through → Regional Performance**.

**Expected output:** the card says **Northeast**; the table lists **24 stores**, led by **NorthStar Newark #123** ($203,125.04), NorthStar New York #111 ($196,707.48), NorthStar Philadelphia #121 ($193,125.98) for FY2026 — the Fiscal Year filter travels with the drill-through. **Back** returns to page 1.

Optional pages 3–5 from the guided lab use measures you already have: **Product Analytics** (matrix category → subcategory → product, `Top 20% Products Revenue %` card), **Customer Analytics** (`customer_segment` slicer, `New Customer Revenue` vs `Repeat Customer Revenue`), **Operational KPIs** (`Revenue by Ship Date`, `Avg Days to Ship`, `Return Rate`).

✅ **Checkpoint:** drill-through from any region lands on a page filtered to that region and the Back button works.

### Step 7 — Row-level security (guided lab Part 4)

**7a. Mapping tables.** Transform data → New Source → **Blank Query** → Advanced Editor → paste → name the query exactly as in the comment. Repeat for the second one. **Close & Apply.** Do **not** relate them to other tables; hide both in Report view.

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

```m
// security_user_store — Maps store managers to their store
let
    SecurityTable = Table.FromRecords({
        [user_email = "boston101.manager@northstar.com",  store_id = 101],
        [user_email = "atlanta125.manager@northstar.com", store_id = 125]
    }),
    TypedTable = Table.TransformColumnTypes(SecurityTable, {
        {"user_email", type text},
        {"store_id",   Int64.Type}
    })
in
    TypedTable
```

**7b. Roles.** **Modeling → Manage roles** → **New** role `Regional Access` → select table `dim_store` → switch to the **DAX editor** (or *Edit in DAX*) → paste → **Save**:

```dax
// Dynamic RLS: the logged-in user's email determines which region they see.
// USERPRINCIPALNAME() returns the user's sign-in name (UPN) in the Service.
// LOOKUPVALUE finds their authorised region in the mapping table.
// "__ALL__" is the CFO's marker: keep every row.
// The filter on dim_store flows to fact_sales AND fact_targets
// through the active single-direction relationships.
VAR UserRegion =
    LOOKUPVALUE(
        security_user_region[region_name],
        security_user_region[user_email],
        USERPRINCIPALNAME()
    )
RETURN
    UserRegion = "__ALL__" || dim_store[region_name] = UserRegion

// IMPORTANT: users not in the mapping table get BLANK from LOOKUPVALUE,
// so the filter keeps no rows (all data hidden). Add every user to the
// mapping before publishing, with exactly one row per email.
```

New role `Store Access` → table `dim_store`:

```dax
// Even more restrictive — store managers see only their own store
dim_store[store_id] =
    LOOKUPVALUE(
        security_user_store[store_id],
        security_user_store[user_email],
        USERPRINCIPALNAME()
    )
```

**How it works:** a role filter is evaluated once per row of `dim_store`; rows where it returns TRUE stay visible. Because the filter is on a **dimension**, it flows along the single-direction relationships to **both** facts — one rule secures `fact_sales` and `fact_targets`. The original guided lab's rule compared `region_name` directly with the lookup result, so the CFO's `"__ALL__"` row matched no store and the CFO saw *nothing*; the `UserRegion = "__ALL__" ||` branch fixes that.

**7c. Test from the outside** — **Modeling → View as**, Executive Summary, Fiscal Year 2026:

| View as (Other user + role) | Expected Net Revenue | Also check |
|---|---|---|
| `ne.manager@northstar.com` + Regional Access | **$3,346,490.01** | Region bars show only Northeast; Revenue Target $3,239,600.00 |
| `cfo@northstar.com` + Regional Access | **$11,862,350.26** | All four regions |
| `boston101.manager@northstar.com` + Store Access | **$80,855.91** | Only *NorthStar Boston #101* |
| `intruder@northstar.com` + Regional Access | **(blank)** | Every visual empty — the negative test |

Note what happens to `Revenue % of Company Total` for the Northeast manager: **100%**. `REMOVEFILTERS` cannot lift a security filter, so "the company" is everything the user may see. Decide with the CFO whether that is wanted (it usually is: managers should not infer other regions' totals).

Click **Stop viewing** when done.

✅ **Checkpoint:** all four rows behave exactly as in the table — especially the blank one.

### Step 8 — Performance pass (guided lab Part 6)

1. **Optimize** ribbon (older versions: **View**) → **Performance Analyzer** → **Start recording** → **Refresh visuals**. Then change the Fiscal Year slicer and click a region bar.
2. **Stop**. Expand each visual: *DAX query* (engine time), *Direct query* (0 — this model is Import), *Other* (rendering/waiting). Sort by duration.

| Duration | Action |
|---|---|
| < 200 ms | Excellent — leave it |
| 200–500 ms | Good — monitor |
| 500 ms – 2 s | Investigate the DAX |
| > 2 s | Must fix before production |

3. For the slowest visual: **Copy query** → paste into **DAX query view** (or DAX Studio with *Server Timings*). Many storage-engine queries for one visual usually mean an iterator walking a large table row by row.
4. Walk the guided lab's checklist and note what this model already does:

| Optimisation | NorthStar model |
|---|---|
| 1. Remove unused columns | `created_by`, `source_system`, `email`, `order_line_id`, `line_number`, `open_date`, `unit_cost` never imported (Lab 02) |
| 2. Integer keys | All keys are whole numbers; dates use `YYYYMMDD` keys |
| 3. Measures over calculated columns | No calculated columns; row-level logic in Power Query |
| 4. Variables in DAX | `YoY Growth %`, `Return Rate`, `Revenue Status`, `Top 20% Products Revenue %` |
| 5. Auto date/time off | Since Lab 00; `dim_date` marked |

5. Compare file sizes:

```powershell
Get-ChildItem C:\DataEngineering\sk06\pbix\*.pbix | Select-Object Name, @{n='MB'; e={[math]::Round($_.Length / 1MB, 1)}}
```

**Expected output:** a table of your lab files and sizes. The model files from Lab 02 onwards hold *more* useful information than Lab 01's naive file (names, dates, costs, targets) for a comparable size, because the fact table is narrow and integer-keyed and the dimensions are small.

6. Write a three-line performance note in your project notes: slowest visual before/after (ms), what you changed, model size.

✅ **Checkpoint:** every visual on the Executive Summary is under 500 ms on a typical laptop, and you have a written before/after note.

### Step 9 — Mobile layout

The CFO "cannot check numbers on her phone" today. **View → Mobile layout** → drag onto the phone canvas, in this order: the Fiscal Year slicer, the four KPI cards (stacked), the Revenue Trend chart, the Revenue by Region chart. Switch back with **View → Mobile layout** again. Published reports open this layout automatically in the Power BI mobile app.

✅ **Checkpoint:** the mobile canvas shows slicer, four cards and two charts, readable without zooming.

### Step 10 — Publishing plan (guided lab Part 7)

Even without a Service account, write the plan; with one, do it.

**What each Service feature needs:**

| Feature | Needs |
|---|---|
| Publish to *My workspace* | A work/school account with Power BI (free licence is enough to publish to *My workspace*) |
| Share, workspaces, apps for others | Pro or Premium Per User for publisher and viewers, or Premium/Fabric capacity |
| **Deployment pipelines** (Dev → Test → Prod) | Premium Per User, Premium or Fabric capacity (a Fabric trial works) |
| **Scheduled refresh** of this model | An **on-premises data gateway** that can reach `localhost:5434` — your container is not on the internet |
| RLS enforcement | Users must be **Viewers** (or app consumers). Workspace Admins/Members/Contributors are **not** filtered by RLS |

**If you have Service access:**

1. **Home → Publish** → *My workspace* (or a workspace you can use).
2. In the Service: semantic model → **⋯ → Security** → add a test account to `Regional Access`. Ask the account to open the report: only its region appears if its email is in `security_user_region` (otherwise everything is blank — the negative test again).
3. Semantic model → **Settings → Data source credentials**: without a gateway it says the source is unreachable — expected for a laptop source.

**Write down (half a page) the guided lab's production design:**

- Workspaces `NorthStar-Dev`, `NorthStar-Test`, `NorthStar-Prod`, assigned to one deployment pipeline. Deployment **parameter rules** set `ServerName` per stage (`northstar-dev-db:5432`, `northstar-test-db:5432`, `northstar-prod-db:5432`) — the reason you created parameters in Lab 02.
- Workflow: developer publishes to Dev → deploy to Test → QA validates numbers against SQL control totals and checks Performance Analyzer → team lead approves → deploy to Prod. Two named people may deploy to Prod.
- Refresh: daily at **05:00** (before the CFO's day), timeout 2 hours, failure email to `data-team@northstar.com`, through gateway `NorthStar-PBI-Gateway`.
- Security: regional managers' group → `Regional Access`; store managers' group → `Store Access`; consumers get an **app** (Viewer rights), never workspace Member rights.

✅ **Checkpoint:** a written plan covering workspaces, pipeline rules, refresh, gateway and security groups (plus a published report if you have access).

### Step 11 — Final check and save

1. Hide helper columns and tables from Report view (keys, `Year-Month Sort`, security tables).
2. Every measure has a description; every visual has a meaningful title.
3. **Ctrl+S**. Optionally also save as `C:\DataEngineering\sk06\pbix\NorthStar_Sales.pbix` — the file name the guided lab uses.

✅ **Checkpoint:** a colleague can open the file cold and (a) get the headline in 5 seconds, (b) drill into their region, (c) see only what their role allows.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Fiscal Year slicer lists 2020–2028 | `dim_date` covers more years than the data | Visual-level filter on the slicer: keep 2024–2026 (or filter `Net Revenue` is not blank). |
| Trend chart months out of order | X-axis uses `Month Name`, or `Year-Month` not sorted | Use `Year-Month` sorted by `Year-Month Sort` (Lab 03 Step 7). |
| Bar colours ignore the status | Rules based on the wrong field, or values typed as text | fx → Rules → *Based on field* `Revenue Status`, numeric rules `= 1`, `= 0`, `= -1`. |
| Top N table shows more than 10 rows | Ties, or the filter was not applied | Click *Apply filter* in the Filters pane; ties at position 10 can add rows. |
| Drill through option missing on right-click | Field in the Drill through well differs from the visual's field | Use `dim_store[region_name]` in both. |
| Map is blank | Map visuals disabled, or lat/long not categorised | Options → Security → *Use Map and Filled Map visuals*; Lab 03 Step 8 data categories. |
| CFO sees nothing under RLS | `"__ALL__"` branch missing (original guided lab bug) | Use the `UserRegion = "__ALL__" ||` rule. |
| *A table of multiple values was supplied where a single value was expected* | Two rows for the same email in a mapping table | Exactly one row per email. |
| Everyone sees everything in the Service | Users are workspace Members/Contributors, or role has no members | Give Viewer access / an app; add users or groups to the role. |
| Refresh fails in the Service | No gateway for the laptop source | Install/configure an on-premises data gateway, or treat refresh as out of scope locally. |

## Key takeaways

- One page, one question; the answer top-left; colour only for the message. Test with five seconds.
- Drill-through lets one report serve the CFO (summary) and the regional manager (detail).
- Secure the **dimension** with a dynamic rule — the filter flows to every fact. Handle the "see everything" user explicitly and always run the negative test.
- Measure before optimising; the NorthStar model already follows all five checklist items because the earlier labs shaped it that way.
- Publishing needs the right licence, a gateway for private sources, and Viewer-only consumers for RLS to apply; deployment pipelines need PPU/Premium/Fabric capacity and use parameter rules.

**Interview check:** *How do you implement dynamic RLS?* — A role with a DAX filter on a dimension comparing a mapping-table lookup with `USERPRINCIPALNAME()`; filters flow along relationships to facts; test with View as, including an unmapped user; enforcement happens in the Service for Viewers. *A report is slow — your approach?* — Performance Analyzer to find the visual and whether time is DAX or rendering; inspect the query; fix the classics (unused columns, iterators over facts, auto date/time, too many visuals); re-measure and document.

## Next

The [SK-06 End-to-End Guided Lab](../Labs/SK-06_End_to_End_Guided_Lab.md): build the whole NorthStar solution again in one file, `NorthStar_Sales.pbix`, from Part 0 to Part 7 — then answer its reflection questions. After that, the [SK-06 Capstone Project](../Project/SK-06_Capstone_Project.md).
