# Lab 06 — Report Design, Dashboards, RLS, Performance & Publishing

> **Module:** SK-06 Power BI Analytics Practitioner
> **Estimated time:** 6–7 hours
> **Prerequisites:** Lab 05 — your NorthStar Retail model has a star schema, a marked date table, core measures, and time-intelligence measures.

---

## 1. Environment Setup

No new installs. Open your Lab 05 `.pbix` and save a copy for this lab:

**File → Save As → `NorthStar_Lab06.pbix`** (versioned copies before big changes — the Desktop equivalent of a git branch).

Verify your starting state:
- Model view shows the star schema (fact in the middle, dimensions around it).
- The date table is **marked as date table** (Table tools → Mark as date table).
- Measures live in a dedicated measure table or well-organized folders.

**Common problems:** if visuals show blank after opening, check Data source settings (File → Options → Data source settings) — moved CSV folders are the usual cause; repoint and Refresh.

---

## 2. Business Context

Everything until now made the model *correct*. This lab makes it *usable, safe, fast, and shippable* — the four things that decide whether a report is adopted or abandoned:

- **Usable:** NorthStar's ops director gives any report **five seconds** — if the headline isn't obvious, she closes it. Report design is not decoration; it's the delivery mechanism for everything you built.
- **Safe:** regional managers must see **only their region's** numbers. One leaked row of another region's margin data is a trust incident (and in HR/finance contexts, a legal one). That's **Row-Level Security (RLS)**.
- **Fast:** a report that takes 20 seconds per click gets abandoned no matter how correct it is. **Performance Analyzer** finds the slow visual before your users do.
- **Shippable:** Desktop is your workshop; the **Power BI Service** is the showroom. Even working Desktop-only in this module, you must *design for* publishing: refresh strategy, workspaces, and app distribution are interview and client staples.

**Who consumes this lab's output?** Every persona at NorthStar: executives (overview page), regional managers (their slice only, via RLS), analysts (drill-through detail). **If you skip this layer:** technically-correct models die in drawer — the most common fate of BI projects, and entirely preventable.

---

## 3. Concept Explanation

### 3.1 Report design: the 5-second rule and the Z-pattern

Readers scan a page like a Z: top-left → top-right → bottom. So:
- **Top-left = the answer**: the single most important KPI (Total Sales vs Target).
- **Top strip = headline cards** (3–5 KPIs, never 10).
- **Middle = the "why"**: one main trend/comparison visual.
- **Bottom/right = the detail**: breakdowns, tables.
- **One page = one question** ("How are sales doing?" / "Why is margin down?"). Multi-question pages answer nothing.

Visual selection heuristics: trend over time → line; comparison across categories → bar (horizontal for long labels); part-to-whole → 100% stacked bar or treemap (pie only with ≤ 4 slices); single number vs target → card + KPI; geographic → map only when geography *is* the question. Fewer visuals, larger, with white space — data-ink over chart-junk.

### 3.2 Interactivity: slicers, drill-through, bookmarks, tooltips

- **Slicers** filter a page; the **filter pane** does the same with less canvas cost — prefer the pane for rarely-changed filters.
- **Drill-through** = right-click a data point → jump to a detail page filtered to that point. This is how one report serves both the 5-second executive and the deep-dive analyst.
- **Bookmarks** capture a page state (filters, visibility) — used for story navigation and toggle buttons (e.g., switch a visual between Sales/Margin).
- **Report page tooltips** — hovering a bar shows a mini-page of context. High polish per minute invested.

### 3.3 Row-Level Security (RLS)

A **role** is a named DAX filter applied to a table, e.g. role *Region Manager – West* filters `dim_region[region] = "West"`. **Dynamic RLS** goes further: one role, one rule — `dim_region[manager_email] = USERPRINCIPALNAME()` — and each signed-in user sees exactly their rows, driven by a mapping table. Rules to burn in:

- RLS filters flow along relationships (secure the *dimension*, the facts follow).
- RLS is enforced in the Service; in Desktop you *test* with **View as**.
- RLS is a **model** feature, not a report feature — every report on the model inherits it. (And it is *security*, unlike hiding visuals, which is decoration.)

### 3.4 Performance: where reports get slow

Order of usual suspects: (1) too many visuals per page (each is ≥ 1 query); (2) high-cardinality iterators in DAX (`FILTER` over a fact table instead of dimension); (3) bi-directional relationships forcing extra work; (4) Auto date/time hidden tables bloating the model; (5) imported columns nobody uses. **Performance Analyzer** measures per-visual query time; **DAX query view** lets you test suspect measures. Measure first, optimize second — the SK-01 profiling discipline, again.

### 3.5 Publishing concepts (Service, even though we build in Desktop)

- **Workspace** = team folder in the Service (dev/test/prod separation).
- **Semantic model** (the published dataset) is *shared*: many reports, one model — why naming and documentation matter.
- **App** = the packaged, read-only distribution to consumers.
- **Refresh**: imported models refresh on schedule (up to 8×/day Pro) via a **gateway** if sources are on-premises.
- **Deployment pipelines** promote content dev→test→prod. You'll live all of this in SK-07 (Fabric); today you design for it.

---

## 4. Step-by-Step Implementation

### Step 1 — Wireframe before you build

**What:** On paper (really), sketch three pages:

1. **Executive Overview** — top-left: Total Sales card with vs-PY %; top strip: Margin %, Units, Avg Order Value; middle: Sales vs PY by month (line); right: Sales by region (bar); bottom: top 10 products (bar).
2. **Region Detail** (drill-through target) — region header, trend, store table, product mix.
3. **Data Quality/About** — last refresh date, record counts, definitions of each measure.

**Why wireframe first:** placing visuals before deciding the page's question guarantees clutter. Professionals sketch; amateurs drag-and-drop-and-regret. The About page is a production touch clients notice — it answers "when was this refreshed and what does 'margin' mean here?" before anyone asks.

**Verify:** each page's question written at the top of the sketch, one sentence.

### Step 2 — Build the Executive Overview

**What:** Build page 1 from your wireframe:

- Insert **cards** for the four KPIs using your Lab 04/05 measures (`Total Sales`, `Sales vs PY %`, `Margin %`, `Total Units`).
- **Line chart**: `Total Sales` and `Sales PY` by `dim_date[Month]` — two series, current year emphasized (thicker line), PY muted grey.
- **Bar chart**: `Total Sales` by `dim_region[region]`, sorted descending.
- **Top-N bar**: `Total Sales` by product, filter pane → Top N = 10 by Total Sales.
- Apply a consistent theme: View → Themes (pick one; customize primary color to a NorthStar blue). Align everything (Format → Align); equal gutters.

**Why the PY series is grey:** color is information — reserve strong color for *the* number, mute the reference. A page where everything shouts says nothing (this single habit visibly separates professional reports).

**Expected result / verify with the 5-second test:** show the page to someone (or squint at it): can they state "sales are up/down vs last year and the West region leads" in five seconds? If not, remove or shrink something — the fix is nearly always subtraction.

**Common mistakes:** ten KPI cards (nobody reads ten); pie chart with twelve slices; titles like "Chart 1" — every visual title should state its message ("Sales up 8% vs PY, driven by West").

### Step 3 — Drill-through and polish

**What:**
1. Build the **Region Detail** page: add `dim_region[region]` to the page's **Drill-through** well (Visualizations pane → Drill through). Add the trend line, a store-level table (store, sales, margin %, units), product mix bar.
2. On the overview's region bar chart: right-click a bar → Drill through → Region Detail — Power BI wired it automatically. A back button appeared top-left (keep it).
3. Add a **report page tooltip**: new page, set Page information → Tooltip = on, size Tooltip; add mini trend + KPI for whatever is hovered; then on the overview bar chart → Format → Tooltip → Report page → select it.
4. **Bookmark toggle**: duplicate the main line chart but showing Margin %; overlap the two; create two bookmarks (each hides one visual via the Selection pane); add two buttons wired to the bookmarks ("Sales" / "Margin").

**Why drill-through beats more pages of slicers:** it matches how humans investigate — see anomaly, click anomaly, get context *about that anomaly*. Zero navigation training required.

**Verify:** right-click → drill-through lands filtered to the clicked region (check the header shows the region name via a `SELECTEDVALUE(dim_region[region])` measure). Tooltip appears on hover. Buttons swap the visual.

### Step 4 — Row-Level Security

**What:**
1. **Static roles first:** Modeling → Manage roles → New: role `Region - West`, table `dim_region`, DAX filter:

   ```dax
   [region] = "West"
   ```

   Create `Region - East` similarly.
2. **Test:** Modeling → **View as** → `Region - West`. Every visual on every page now shows West only. Check the overview cards recalculated. Stop viewing.
3. **Dynamic RLS:** create a mapping table (Enter data or CSV): `manager_email`, `region` — a few rows, including your own email → West. Relate it to `dim_region` (region → region). New role `Regional Managers` on the mapping table:

   ```dax
   [manager_email] = USERPRINCIPALNAME()
   ```

4. **Test:** View as → check both `Regional Managers` **and** "Other user" = your mapped email. You see West only.

**Why secure the dimension:** the filter flows across the relationship to every fact row automatically — one rule secures the whole model. Filtering the fact table directly is fragile (misses new facts) and slow (millions of rows evaluated).

**Common mistakes:**
- Forgetting RLS only *enforces* in the Service — in Desktop everyone with the file sees everything. The `.pbix` itself must be handled as sensitive.
- Bi-directional relationships + RLS = filters can flow *around* your security. Keep single-direction unless you can argue otherwise.
- Testing only the happy role. Also test a user *not* in the mapping table — they should see **nothing** (verify this; "sees everything" means your rule has a hole).

### Step 5 — Performance pass

**What:**
1. **Measure:** Optimize ribbon → **Performance Analyzer** → Start recording → Refresh visuals. Sort by duration. Anything > 1,000 ms goes on the fix list. Copy the slowest visual's DAX query (it's a button) and inspect it in **DAX query view**.
2. **Fix the classics:**
   - File → Options → Data Load → **untick Auto date/time** (then confirm your own date table still drives time intelligence). Model size drops immediately — check File size before/after.
   - Power Query: remove columns no visual or measure uses (Choose Columns). Every imported column costs memory forever.
   - Any measure with `FILTER('fact_sales', ...)` inside CALCULATE: rewrite the condition against a dimension or as a simple predicate (`CALCULATE([Total Sales], dim_product[category] = "Toys")`) and re-measure.
   - Reduce visuals per page if a page runs > 10 queries.
3. **Re-measure** and record before/after numbers.

**Why "measure first" is the whole method:** the slow thing is never what you'd guess (it's usually one iterator over the fact table, or simply too many visuals). Optimizing without Performance Analyzer numbers is superstition; with numbers it's engineering — and the before/after table is capstone evidence.

**Expected outcome:** total page refresh time visibly reduced; document the numbers (e.g., "overview page 4.2 s → 1.1 s; removed auto date/time, rewrote 2 iterator measures, cut 14 unused columns").

### Step 6 — Prepare for publishing (design, document, decide)

**What:** Even Desktop-only, produce the shipping artifacts:

1. **Refresh strategy note** (half a page, goes with your project docs): source = CSV folder → in production would be a gateway-connected folder/database; schedule = daily 06:00 before the 08:00 ops review; failure notification = email to the analytics team (Service setting). Import mode justified (data ≤ millions of rows, sub-second interaction beats DirectQuery latency; revisit if near-real-time is ever required).
2. **Model documentation:** every measure gets a **description** (Model view → select measure → Description field) — the Service surfaces these as tooltips to end users. Hide from Report view every column users shouldn't touch (keys, raw columns superseded by measures) — a curated field list *is* documentation.
3. **Workspace plan** (one paragraph): dev workspace (you), test (the ops director validates), prod app (regional managers, RLS enforced). Names, owners, promotion cadence.
4. If you have a work/school account with Fabric/Power BI Service access: **Publish** (Home → Publish) to your My Workspace and verify RLS: Service → semantic model → Security → map a colleague/test account into `Regional Managers`. If you don't have Service access, the strategy documents above are the deliverable — and SK-07 gives you the full Service experience.

**Why descriptions and hiding matter:** the published semantic model is a *product for self-service users* — a tidy, documented field list is the difference between "the model answers questions" and "fifty Teams messages asking what gross_amt_2 means."

**Verify:** a colleague (or you, tomorrow) can open the report cold and (a) get the headline in 5 s, (b) drill to their region, (c) find what any measure means without asking.

---

## 5. Production Engineering Practices Introduced in This Lab

| Practice | Why it matters |
|---|---|
| **Wireframe before building; one page = one question** | Design intent beats visual accretion; the 5-second test is the acceptance criterion. |
| **Color as information (emphasize the answer, mute the reference)** | Professional-grade visual communication, instantly visible. |
| **Drill-through architecture (overview → detail)** | One report serves executives and analysts without training. |
| **RLS on dimensions; dynamic via USERPRINCIPALNAME + mapping table** | Security that scales with users, not with roles; flows to every fact automatically. |
| **Test RLS with View-as, including the unmapped-user case** | Security you haven't tested from the outside isn't security. |
| **Performance Analyzer before optimizing; documented before/after** | Measurement-driven tuning — the profiling discipline in BI clothing. |
| **Measure descriptions + hidden raw columns** | The semantic model as a documented product for self-service. |
| **Written refresh/workspace/distribution strategy** | Desktop-only today, but designed for production — exactly what clients ask to see. |

---

## 6. Reflection

### What you learned
Turning a correct model into a shippable product: designed pages that pass the 5-second test, drill-through and polish, static and dynamic RLS with honest testing, a measured performance pass, and the publishing/refresh strategy that makes it production-real.

### Why it matters
Adoption is the metric BI lives or dies by. Models don't get adopted; *experiences* do — fast, safe, obvious ones. This lab is also the direct rehearsal for the CloudHR capstone, where RLS ("I see my company only") and the 5-second test are explicit client acceptance criteria.

### Interview questions (with model answers)

1. **"How do you implement row-level security in Power BI?"**
   Roles with DAX filters on dimension tables; dynamically via `USERPRINCIPALNAME()` against a user-mapping table related to the dimension. Filters flow along relationships to facts. Test with View-as including unmapped users; enforcement happens in the Service.

2. **"A report is slow. Walk me through your approach."**
   Performance Analyzer to find *which* visual and whether time is DAX or rendering; inspect the query in DAX query view; usual fixes: kill Auto date/time, remove unused columns, rewrite fact-table iterators to dimension predicates, reduce visuals per page. Re-measure and document before/after.

3. **"How do you design a report page?"**
   One question per page; Z-pattern with the answer top-left; 3–5 KPI cards; one main explanatory visual; drill-through for detail; color reserved for the message. Acceptance test: a stakeholder gets the headline in five seconds.

4. **"Import vs DirectQuery?"**
   Import: data cached in the model — fastest interaction, scheduled refresh, the default. DirectQuery: queries hit the source live — near-real-time and no-copy compliance cases, at latency/feature cost. Choose by freshness requirement and volume; say the trade-off, not a favorite.

5. **"What's your process for publishing and refreshing a report?"**
   Dev/test/prod workspaces, promote via deployment pipeline; app for consumers; scheduled refresh (gateway if on-prem sources) timed before the business's consumption moment; refresh-failure alerts; RLS mappings maintained in the Service.

6. **"Why hide columns and write measure descriptions?"**
   The published model is a self-service product — a curated, documented field list prevents misuse (raw columns vs measures) and support load. Descriptions surface as tooltips to end users.

### Common interview traps
- Treating RLS as report-level (visual hiding) — it's model-level security; hiding is cosmetics.
- Optimizing DAX before measuring — the expected first words are "Performance Analyzer."
- Pie charts and 12-KPI dashboards in your portfolio screenshots — reviewers notice design maturity instantly.
- Not knowing that Desktop doesn't enforce RLS — the `.pbix` is the keys to everything.

### Key takeaways
1. One page, one question; the answer top-left; five seconds.
2. Secure dimensions dynamically; test from the outside, including the nobody-user.
3. Measure, fix the iterator/auto-date/unused-column classics, re-measure, document.
4. Ship a *product*: descriptions, hidden raw fields, refresh strategy, workspace plan.

**Next:** the [CloudHR capstone](../Project/01_Business_Scenario.md) — new client, new domain (HR analytics), same discipline, harder data.
