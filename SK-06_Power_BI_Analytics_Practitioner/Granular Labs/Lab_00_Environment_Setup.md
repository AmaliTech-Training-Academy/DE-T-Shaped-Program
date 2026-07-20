# Lab 00: Environment Setup

> **Module:** SK-06 Power BI Analytics Practitioner
> **Estimated time:** 2–3 hours
> **Previous:** [../README.md](../README.md) (module overview)
> **Next:** [Lab_01_Power_BI_and_BI_Concepts.md](Lab_01_Power_BI_and_BI_Concepts.md)

Welcome to the first lab of SK-06. This lab does the unglamorous but critical work: installing Power BI Desktop the right way, configuring the settings that most beginners never touch (and later regret), building a clean folder structure, and generating the **NorthStar Retail** dataset you will use in every subsequent lab.

Do not skip or skim this lab. Roughly a third of "Power BI is broken" complaints from beginners trace back to something this lab prevents.

---

## Table of Contents

1. [Environment Setup](#1-environment-setup)
2. [Business Context](#2-business-context)
3. [Concept Explanation](#3-concept-explanation)
4. [Step-by-Step Implementation](#4-step-by-step-implementation)
5. [Production Engineering Practices](#5-production-engineering-practices)
6. [Reflection](#6-reflection)

---

## 1. Environment Setup

This entire lab *is* environment setup, so this section covers requirements and installation, and Section 4 walks through everything step by step with verification.

### 1.1 What you need

| Requirement | Detail |
|---|---|
| Operating system | Windows 10 or Windows 11 (64-bit). This module assumes **Windows 11**. |
| RAM | 8 GB recommended (4 GB absolute minimum — Power BI holds your data model in memory) |
| Disk | ~2 GB free (app + lab data) |
| Account | None required for this module. A work/school email is only needed if you later sign in to the Power BI Service — we stay in Desktop. |
| Cost | **$0.** Power BI Desktop is completely free. |

> **macOS users — read this now.** Power BI Desktop is **Windows-only**. There is no Mac version and no plan for one. Your options: (a) a Windows 11 VM via Parallels or VMware Fusion, (b) a cloud Windows VM (e.g., an AWS EC2 Windows instance — you know how from SK-04), or (c) a spare Windows machine. Give the VM at least 8 GB RAM. Everything else in this module works identically inside the VM.

### 1.2 Two ways to install Power BI Desktop

You will pick ONE of these in Section 4. Both are official and free; they differ in how updates happen.

| | Option A: Microsoft Store (recommended) | Option B: Direct download (.exe installer) |
|---|---|---|
| Where | Microsoft Store app, search "Power BI Desktop" | `https://www.microsoft.com/download` → search "Power BI Desktop" |
| Updates | **Automatic** — the Store updates it in the background | **Manual** — you download and rerun the installer each month |
| Admin rights needed | No | Yes (typical corporate blocker) |
| Best for | Personal machines, this course | Locked-down corporate machines, or when IT pins a specific version |

**Version guidance:** Power BI Desktop ships a new release **every month**. Always use the **latest stable** version — there is no "LTS" for Desktop, and Microsoft only supports recent versions for publishing. You check your version via **Help > About** (covered in Step 3 below). During this module, if a ribbon button is in a slightly different place than described, a monthly update probably moved it — the concepts never change that fast.

### 1.3 Optional tools (install later, not today)

Two free community-standard tools are referenced in later labs. **Do not install them now** — just know they exist so the names aren't alien later:

- **DAX Studio** (free, `daxstudio.org`) — a query and performance tool for the model inside your .pbix. Used optionally in Lab_04–Lab_06 for measuring query speed.
- **Tabular Editor 2** (free, `tabulareditor.com`) — an external model editor that lets you create/edit many measures at once. Used optionally from Lab_03 onward. (Tabular Editor **3** is paid; version **2** is free and is what we mean.)

### 1.4 Folder structure

All lab work lives under `C:\PowerBI_Labs\` — created by script in Section 4. Keeping lab files out of `Downloads` and OneDrive-synced folders avoids two classic problems: files auto-syncing mid-save (corrupting .pbix files) and "I can't find my data source" path chaos.

---

## 2. Business Context

### The problem this lab solves

Meet **NorthStar Retail**: a mid-market retail chain with **85 stores** across **4 regions**, selling **12,000 SKUs** in **5 product categories**. (A *SKU* — stock-keeping unit — is simply a distinct sellable product; a red T-shirt in size M is a different SKU from the same shirt in size L.)

Every Monday morning, an analyst named Kwame spends 6+ hours exporting data, copying it into Excel, fixing broken cell references, and emailing a **40-page report** to the CFO. By the time she reads it, the data is a week old. Twice last quarter, a copy-paste error made a region's revenue look 30% lower than reality, and a district manager got an angry phone call for nothing.

Your mission across Labs 01–06 is to replace that report with an interactive Power BI dashboard. But before you can analyze anything, you need two things: a working tool, and data. That is this lab.

### Why environment setup matters in industry

In a real job, "install the tool" is rarely trivial:

- **IT restrictions** decide whether you can use the Store or need the .exe with an admin ticket.
- **Version drift** across a team ("works on my machine, my Power BI is from 2024") causes files that open with warnings or subtle behavioral differences.
- **Default settings** — especially **Auto date/time**, which we disable in this lab — silently bloat models. Consultants regularly find client models where *half the file size* is invisible auto-generated date tables.
- **Test data generation** is a genuine professional skill. You cannot practice on production data (privacy, access), so engineers create *synthetic data* — realistic but fake. Our PowerShell generation script even injects deliberate data-quality problems, because real data is never clean and you need something to fix in Lab_02.

### Who consumes the output of this lab?

You do — every later lab assumes this environment exactly as built here. In a company, the "consumer" of an environment-setup document is *the next new hire*: a good setup guide turns a week of onboarding friction into an afternoon.

### What happens if this goes wrong?

- Skip disabling Auto date/time → your Lab_03 model carries seven hidden date tables, your file is several times larger than needed, and your Lab_06 performance tuning fights ghosts.
- Put files in a OneDrive folder → a sync conflict one day corrupts your only copy of the model.
- Generate data without a fixed random seed → your numbers won't match the lab checkpoints and you can't tell whether *you* made a mistake or the data is just different.

---

## 3. Concept Explanation

Even a setup lab has concepts worth understanding rather than cargo-culting.

### 3.1 What is Power BI Desktop, actually?

**Power BI Desktop** is a free Windows application that bundles three engines into one tool:

1. **Power Query** (the "mashup engine") — connects to data sources and transforms data. Think of it as visual ETL: the same extract-transform-load work you scripted in SK-03, driven through a UI that writes code (in a language called **M**) for you.
2. **The VertiPaq storage engine** — an in-memory, **columnar** database that compresses your data and holds it in RAM. *Columnar* means data is stored column-by-column instead of row-by-row, which compresses beautifully when a column has repeated values (a `region` column with 4 distinct values across 50,000 rows compresses to almost nothing). This is the same family of idea behind Parquet from SK-03/SK-04.
3. **The DAX formula/query engine** — evaluates calculations (measures) written in **DAX** (Data Analysis Expressions) against the model.

Everything you build gets saved into a single **`.pbix` file** — a zip-like container holding your queries, model, data, and report pages. (Lab_01 dissects a .pbix in detail.)

**Alternatives, for perspective:** Tableau (strong visuals, separate prep tool, more expensive), Looker (code-first semantic layer, Google-owned), Apache Superset / Metabase (open source, lighter modeling), plain Excel (ubiquitous but breaks at scale — see: Kwame's Mondays). Power BI's pros: price (free Desktop, cheap licensing bundled with Microsoft 365), one integrated tool for prep+model+visuals, huge job market. Cons: Windows-only authoring, DAX has a real learning curve, and the best sharing features require paid cloud licensing.

### 3.2 Why "latest version, updated monthly" is the rule

Power BI Desktop has no long-term-support versions. Microsoft releases monthly; features preview, then generalize. A .pbix saved in a newer version may not open in an older one (**forward compatibility is not guaranteed**), which is exactly why teams standardize on "everyone updates monthly." The Microsoft Store install makes this automatic — the main reason it is our recommended path.

### 3.3 What Auto date/time is, and why we turn it off

By default, for **every date/datetime column** in your model, Power BI silently creates a **hidden date table** (with year/quarter/month/day columns) so drag-and-drop date hierarchies "just work" for absolute beginners.

Sounds friendly. Here's the tax:

- **Bloat:** NorthStar's `sales.csv` alone has `order_date` and `ship_date` — two hidden tables. Add dimension dates and you can end up with many hidden tables covering every year appearing anywhere in the data. In real models this is routinely 30–60% of file size.
- **Wrong calendar:** the hidden tables use the **calendar year (Jan–Dec)**. NorthStar's **fiscal year starts July 1**. Every auto-hierarchy "Year" would be *wrong for this business*, invisibly.
- **Broken habits:** professionals always build **one explicit date dimension** (`dim_date`, built in Lab_03) that all facts share. Auto date/time actively hides the need for it.

So we disable it globally today, before you ever load data. This single checkbox is arguably the most important setting in this lab.

### 3.4 Synthetic data and seeded randomness

The generation script uses PowerShell's `Get-Random` with a **fixed seed**. A *seed* initializes the pseudo-random number generator so it produces the *same* "random" sequence every run. Why it matters: reproducibility. Your dataset matches the lab checkpoints, your classmate's dataset matches yours, and rerunning the script after a mistake gives identical data. (Same principle as seeding `random` in SK-01 Python tests.)

The script also plants three defects on purpose — **null customer IDs** (guest checkouts / capture failures), **negative quantities** (returns entered as sales), and **mixed-case region names** (`North` vs `NORTH` vs `north`, classic multi-system inconsistency). Lab_02 exists to find and fix these. Meeting dirty data in Lab_02 with tools, rather than in production with a pager, is the point.

---

## 4. Step-by-Step Implementation

Work through these in order. Each step tells you **what**, **why**, **expected output**, **how to verify**, **common mistakes**, and **troubleshooting**.

### Step 1 — Check your system

**What:** Confirm Windows version and free resources. Open **PowerShell** (press `Win`, type `powershell`, Enter — no admin needed) and run:

```powershell
# OS version, RAM, and free space on C:
Get-ComputerInfo -Property OsName, OsVersion, CsTotalPhysicalMemory |
    Format-List
Get-PSDrive C | Select-Object Used, Free
```

**Why:** Power BI Desktop requires 64-bit Windows and holds models in RAM; better to find a constraint now than mid-lab.

**Expected output:** `OsName` shows Windows 10/11; `CsTotalPhysicalMemory` ≥ ~8 GB (8589934592 bytes); `Free` on C: ≥ 2 GB.

**Verify:** the numbers meet the table in Section 1.1.

**Common mistakes:** Reading `CsTotalPhysicalMemory` as GB — it's bytes. Divide by 1GB mentally or run `(Get-ComputerInfo).CsTotalPhysicalMemory / 1GB`.

**Troubleshooting:** On macOS/Linux you cannot run this at all — set up your Windows VM first (Section 1.1 note), then continue *inside* the VM.

---

### Step 2 — Install Power BI Desktop

**What:** Choose ONE option.

**Option A — Microsoft Store (recommended):**

1. Press `Win`, type `Microsoft Store`, Enter.
2. Search for **Power BI Desktop** (publisher: Microsoft Corporation).
3. Click **Get** / **Install**. Wait for completion (~500 MB download).

Or install from PowerShell using winget (ships with Windows 11):

```powershell
winget install --id Microsoft.PowerBI --source winget --accept-package-agreements
```

> Note: the winget package installs the .exe distribution; installing via the Store app itself is what gives you auto-updates. If auto-updates matter to you (they should), use the Store UI.

**Option B — Direct download:**

1. Browse to `https://aka.ms/pbidesktopstore` fails? Use `https://www.microsoft.com/en-us/download/details.aspx?id=58494`.
2. Download `PBIDesktopSetup_x64.exe` (64-bit).
3. Run it, accept defaults, finish. Requires local admin rights.

**Why:** Store = automatic monthly updates, no admin. Direct = works under corporate IT policies and lets you archive a specific installer version.

**Expected output:** "Power BI Desktop" appears in the Start menu.

**Verify:** Launch it. First run may show a sign-in prompt — **you can close/skip it**; no account is needed for this module. You should land on a blank report canvas (possibly behind a welcome dialog — close the dialog).

**Common mistakes:**
- Installing 32-bit from an old link — always x64.
- Installing "Power BI" the *mobile-style viewer app* from the Store instead of "Power BI **Desktop**". Check the name exactly.

**Troubleshooting (common install problems + fixes):**

| Problem | Fix |
|---|---|
| Store button greyed out / Store blocked by IT | Use Option B, or ask IT to deploy it; mention it needs monthly updates |
| .exe install fails with "administrator privileges required" | Right-click → Run as administrator, or get IT to elevate |
| Error `WebView2 Runtime` missing/failed | Install "Microsoft Edge WebView2 Runtime" from Microsoft's site, then reinstall — Power BI's UI depends on it |
| App launches then immediately closes | Update GPU drivers; if on a VM, enable 3D acceleration or add `DisableGPUAcceleration` via Options once you can open it |
| Extremely slow first launch | Normal once (it compiles caches); subsequent launches are faster |
| Sign-in loop on a corporate machine | Skip sign-in entirely — Desktop authoring never requires it |

---

### Step 3 — Check your version (Help > About)

**What:** In Power BI Desktop: **File > Help > About** (in current releases: **File**, then **About** under Help; or the `?` Help ribbon tab > **About**).

**Why:** You'll be asked "what version are you on?" constantly — by lab troubleshooting notes, forums, and colleagues. Versions are named by month, e.g. `2.14x.xxxx.xxxx (25.06)` style strings where the marketing name is like "June 2025".

**Expected output:** A dialog showing **Version** and **release month**. If it's more than ~2 months old, update (Store: Library > Get updates; Direct: download the newest installer).

**Verify:** Note your version somewhere — you'll record it again in Step 8's verification file.

**Common mistakes:** None serious. Just know where this dialog lives.

**Troubleshooting:** If About shows an old version right after a Store install, the Store cached a stale package — open Microsoft Store > Library > Get updates.

---

### Step 4 — Configure critical options (including disabling Auto date/time)

**What:** In Power BI Desktop go to **File > Options and settings > Options**, then change:

1. **Global > Data Load** → under *Time intelligence*, **uncheck "Auto date/time for new files"**. ← the big one; see Section 3.3 for the full why. Global means every *future* file starts clean.
2. **Current File > Data Load** → uncheck **"Auto date/time"** too (belt and braces for the currently open file).
3. **Global > Report settings** (optional but recommended) → check any available option to disable default interaction surprises later; leave defaults if unsure.
4. **Global > Preview features** → leave everything **as-is**. Preview features change monthly; the labs use only generally-available features so instructions stay stable.
5. Click **OK**.

**Why:** Auto date/time bloats models and imposes a Jan–Dec calendar on a July-fiscal-year business (Section 3.3). Doing it now, globally, means you never have to remember it again.

**Expected output:** Options dialog closes without error.

**Verify:** Reopen Options → Global > Data Load → the Auto date/time checkbox is unchecked and stays unchecked.

**Common mistakes:**
- Unchecking only **Current File** and not **Global** — next lab's new file re-enables it silently.
- Confusing this with "Auto detect new relationships" (a different Data Load option — leave that at default for now; Lab_03 discusses it).

**Troubleshooting:** If the setting seems to "come back," you almost certainly changed Current File only. Global is the top section of the left-hand list in the Options dialog.

---

### Step 5 — Create the lab folder structure

**What:** In PowerShell:

```powershell
# Create the SK-06 working folders
New-Item -ItemType Directory -Force -Path `
    "C:\PowerBI_Labs\NorthStar\data",
    "C:\PowerBI_Labs\NorthStar\pbix",
    "C:\PowerBI_Labs\NorthStar\scripts" | Out-Null

Get-ChildItem -Recurse "C:\PowerBI_Labs" | Select-Object FullName
```

**Why:** One predictable root, outside OneDrive/Downloads (Section 1.4). `data` holds CSVs, `pbix` holds versioned Power BI files, `scripts` holds this lab's generator script.

**Expected output:** Three folder paths listed.

**Verify:** `Test-Path C:\PowerBI_Labs\NorthStar\data` returns `True`.

**Common mistakes:** Creating the folders inside OneDrive's redirected Desktop/Documents. `C:\PowerBI_Labs` at the drive root avoids redirection entirely.

**Troubleshooting:** "Access denied" at `C:\` root is rare but possible on locked-down machines — fall back to `C:\Users\<you>\PowerBI_Labs` and mentally substitute that path in every later lab (or better: ask IT).

---

### Step 6 — Generate the NorthStar dataset

**What:** This is the heart of the lab. Save the following as `C:\PowerBI_Labs\NorthStar\scripts\Generate-NorthStarData.ps1` (create the file in Notepad/VS Code and paste, or type it — pasting is acceptable for this one), then run it.

The script generates six CSVs into `C:\PowerBI_Labs\NorthStar\data`:

| File | ~Rows | Becomes (Lab_03) |
|---|---|---|
| `stores.csv` | 85 | `dim_store` (store → district → region) |
| `products.csv` | 500 | `dim_product` (product → subcategory → category) |
| `customers.csv` | 2,000 | `dim_customer` |
| `promotions.csv` | 12 | `dim_promotion` |
| `sales.csv` | 50,000 | `fact_sales` (grain: one row per order **line item**) |
| `targets.csv` | 96 | `fact_targets` (monthly revenue target per region) |

> Yes, real NorthStar has 12,000 SKUs and millions of sales rows. We generate a 500-SKU / 50,000-row *representative sample* so everything stays fast on a laptop. The modeling techniques are identical at any scale.

It deliberately injects **~2% null customer_ids**, **~1% negative quantities (returns)**, and **mixed-case region names** in stores.csv — Lab_02's cleaning material.

```powershell
<#
.SYNOPSIS
    Generates the NorthStar Retail synthetic dataset for SK-06.
.NOTES
    Seeded (deterministic): rerunning produces identical data.
    Deliberate data-quality issues are injected for Lab_02:
      - ~2% of sales rows have a null customer_id
      - ~1% of sales rows have negative quantity (returns)
      - region names in stores.csv use inconsistent casing
#>

$ErrorActionPreference = 'Stop'
$dataDir = 'C:\PowerBI_Labs\NorthStar\data'
New-Item -ItemType Directory -Force -Path $dataDir | Out-Null

# Fixed seed => reproducible output. Do not change 4242 or your
# numbers will not match the lab checkpoints.
Get-Random -SetSeed 4242 | Out-Null

Write-Host 'Generating NorthStar data (seed 4242)...' -ForegroundColor Cyan

# ---------------------------------------------------------------
# 1. STORES: 85 stores, 4 regions, districts under regions
#    NOTE: region casing is intentionally inconsistent (Lab_02!)
# ---------------------------------------------------------------
$regionVariants = @{
    'North' = @('North','NORTH','north')
    'South' = @('South','SOUTH','south')
    'East'  = @('East','EAST','east')
    'West'  = @('West','WEST','west')
}
$regions   = @('North','South','East','West')
$cities    = @('Springfield','Riverton','Lakeview','Fairmont','Oakdale',
               'Brookside','Hillcrest','Maplewood','Cedarburg','Ashford',
               'Kingsport','Elmwood','Granville','Westbrook','Northgate')
$formats   = @('Flagship','Standard','Standard','Standard','Outlet')

$stores = for ($i = 1; $i -le 85; $i++) {
    $region   = $regions[($i - 1) % 4]
    # 5-6 districts per region, e.g. North-D1..North-D6
    $district = '{0}-D{1}' -f $region, ((Get-Random -Minimum 1 -Maximum 7))
    # Inject casing inconsistency on ~40% of rows
    $regionOut = if ((Get-Random -Minimum 1 -Maximum 101) -le 40) {
        $regionVariants[$region] | Get-Random
    } else { $region }
    [pscustomobject]@{
        store_id     = 'ST{0:d3}' -f $i
        store_name   = '{0} {1}' -f ($cities | Get-Random), ($formats | Get-Random)
        district     = $district
        region       = $regionOut
        open_date    = (Get-Date '2015-01-01').AddDays((Get-Random -Minimum 0 -Maximum 2555)).ToString('yyyy-MM-dd')
        square_feet  = Get-Random -Minimum 8000 -Maximum 45000
    }
}
$stores | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'stores.csv')
Write-Host ('  stores.csv      : {0,6} rows' -f $stores.Count)

# ---------------------------------------------------------------
# 2. PRODUCTS: ~500 products, 5 categories, subcategories
# ---------------------------------------------------------------
$catalog = @{
    'Apparel'        = @('Shirts','Trousers','Outerwear','Footwear')
    'Electronics'    = @('Audio','Mobile Accessories','Computing','Smart Home')
    'Home & Garden'  = @('Kitchen','Furniture','Decor','Garden Tools')
    'Grocery'        = @('Snacks','Beverages','Pantry','Frozen')
    'Sports & Toys'  = @('Fitness','Outdoor','Board Games','Toys')
}
$adjectives = @('Classic','Premium','Eco','Ultra','Compact','Deluxe','Everyday','Pro')
$products = @()
$p = 0
foreach ($cat in ($catalog.Keys | Sort-Object)) {
    foreach ($sub in $catalog[$cat]) {
        # 25 products per subcategory => 5 cats x 4 subs x 25 = 500
        for ($j = 1; $j -le 25; $j++) {
            $p++
            $cost = [math]::Round((Get-Random -Minimum 200 -Maximum 15000) / 100, 2)
            $products += [pscustomobject]@{
                product_id   = 'P{0:d4}' -f $p
                product_name = '{0} {1} #{2}' -f ($adjectives | Get-Random), $sub, $j
                subcategory  = $sub
                category     = $cat
                unit_cost    = $cost
                list_price   = [math]::Round($cost * ((Get-Random -Minimum 130 -Maximum 220) / 100), 2)
            }
        }
    }
}
$products | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'products.csv')
Write-Host ('  products.csv    : {0,6} rows' -f $products.Count)

# ---------------------------------------------------------------
# 3. CUSTOMERS: ~2000 loyalty-program customers
# ---------------------------------------------------------------
$firstNames = @('Ama','Kwame','Efua','Kofi','Sarah','James','Linda','David',
                'Grace','Michael','Akosua','Yaw','Esi','Daniel','Rita','Paul',
                'Naomi','Victor','Abena','Samuel')
$lastNames  = @('Mensah','Owusu','Johnson','Smith','Boateng','Brown','Asante',
                'Williams','Osei','Davis','Appiah','Miller','Addo','Wilson',
                'Agyeman','Taylor','Ansah','Clark','Frimpong','Lewis')
$tiers = @('Bronze','Bronze','Bronze','Silver','Silver','Gold')

$customers = for ($i = 1; $i -le 2000; $i++) {
    [pscustomobject]@{
        customer_id   = 'C{0:d5}' -f $i
        customer_name = '{0} {1}' -f ($firstNames | Get-Random), ($lastNames | Get-Random)
        loyalty_tier  = $tiers | Get-Random
        signup_date   = (Get-Date '2020-01-01').AddDays((Get-Random -Minimum 0 -Maximum 1900)).ToString('yyyy-MM-dd')
        home_region   = $regions | Get-Random
    }
}
$customers | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'customers.csv')
Write-Host ('  customers.csv   : {0,6} rows' -f $customers.Count)

# ---------------------------------------------------------------
# 4. PROMOTIONS: 12 promotions over the two fiscal years
# ---------------------------------------------------------------
$promoNames = @('Back to School','Holiday Blowout','New Year Fresh Start',
                'Spring Refresh','Summer Clearance','Anniversary Sale')
$promotions = @()
$promoId = 0
foreach ($year in @(2023, 2024)) {
    foreach ($pn in $promoNames) {
        $promoId++
        $start = (Get-Date ('{0}-07-01' -f $year)).AddDays((Get-Random -Minimum 0 -Maximum 330))
        $promotions += [pscustomobject]@{
            promotion_id   = 'PR{0:d2}' -f $promoId
            promotion_name = '{0} {1}' -f $pn, $start.Year
            discount_pct   = @(5, 10, 15, 20, 25) | Get-Random
            start_date     = $start.ToString('yyyy-MM-dd')
            end_date       = $start.AddDays((Get-Random -Minimum 7 -Maximum 21)).ToString('yyyy-MM-dd')
        }
    }
}
$promotions | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'promotions.csv')
Write-Host ('  promotions.csv  : {0,6} rows' -f $promotions.Count)

# ---------------------------------------------------------------
# 5. SALES: 50,000 order line items, 2023-07-01 .. 2025-06-30
#    Grain: one row per order line item.
#    Injected issues: null customer_id (~2%), negative qty (~1%)
# ---------------------------------------------------------------
$startDate = Get-Date '2023-07-01'
$daySpan   = 730   # two fiscal years
$rows      = 50000
$sales     = New-Object System.Collections.Generic.List[object]

for ($i = 1; $i -le $rows; $i++) {
    $orderDate = $startDate.AddDays((Get-Random -Minimum 0 -Maximum $daySpan))
    $product   = $products[(Get-Random -Minimum 0 -Maximum $products.Count)]

    # ~2% missing customer_id (guest checkout / capture failure)
    $custId = if ((Get-Random -Minimum 1 -Maximum 101) -le 2) { $null }
              else { 'C{0:d5}' -f (Get-Random -Minimum 1 -Maximum 2001) }

    # ~1% returns recorded as negative quantities
    $qty = if ((Get-Random -Minimum 1 -Maximum 101) -le 1) {
               -1 * (Get-Random -Minimum 1 -Maximum 4)
           } else { Get-Random -Minimum 1 -Maximum 6 }

    # ~25% of lines carry a promotion
    $promo = if ((Get-Random -Minimum 1 -Maximum 101) -le 25) {
                 'PR{0:d2}' -f (Get-Random -Minimum 1 -Maximum 13)
             } else { $null }

    $discount = if ($promo) { @(0.05, 0.10, 0.15, 0.20, 0.25) | Get-Random } else { 0 }

    $sales.Add([pscustomobject]@{
        order_id     = 'SO{0:d7}' -f (($i + 999) - ((Get-Random -Minimum 0 -Maximum 3)))  # some multi-line orders
        order_date   = $orderDate.ToString('yyyy-MM-dd')
        ship_date    = $orderDate.AddDays((Get-Random -Minimum 1 -Maximum 8)).ToString('yyyy-MM-dd')
        store_id     = 'ST{0:d3}' -f (Get-Random -Minimum 1 -Maximum 86)
        product_id   = $product.product_id
        customer_id  = $custId
        quantity     = $qty
        unit_price   = $product.list_price
        discount     = $discount
        promotion_id = $promo
    })

    if ($i % 10000 -eq 0) { Write-Host ('    ...{0} sales rows' -f $i) }
}
$sales | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'sales.csv')
Write-Host ('  sales.csv       : {0,6} rows' -f $sales.Count)

# ---------------------------------------------------------------
# 6. TARGETS: monthly revenue target per region (4 x 24 months)
# ---------------------------------------------------------------
$targets = @()
foreach ($region in $regions) {
    $base = Get-Random -Minimum 400000 -Maximum 700000
    for ($m = 0; $m -lt 24; $m++) {
        $month = (Get-Date '2023-07-01').AddMonths($m)
        $targets += [pscustomobject]@{
            region         = $region
            target_month   = $month.ToString('yyyy-MM-01')
            revenue_target = [math]::Round($base * (1 + $m * 0.01) *
                                 ((Get-Random -Minimum 90 -Maximum 111) / 100), 0)
        }
    }
}
$targets | Export-Csv -NoTypeInformation -Path (Join-Path $dataDir 'targets.csv')
Write-Host ('  targets.csv     : {0,6} rows' -f $targets.Count)

Write-Host 'Done. Data written to' $dataDir -ForegroundColor Green
```

Run it:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass   # this session only
& 'C:\PowerBI_Labs\NorthStar\scripts\Generate-NorthStarData.ps1'
```

**Why:** Real projects never hand you clean CSVs; generating your own (with a seed, with known defects) gives you a controlled, reproducible sandbox — and PowerShell practice on the platform you'll administer Power BI on.

**Expected output:** Progress lines, then six row counts: stores 85, products 500, customers 2000, promotions 12, sales 50000, targets 96, ending with `Done.` The sales generation takes a minute or two.

**How to verify:** Next step is a dedicated verification. Quick sanity check now:

```powershell
Get-ChildItem C:\PowerBI_Labs\NorthStar\data | Select-Object Name, Length
```

All six files present; `sales.csv` should be roughly 4–6 MB.

**Common mistakes:**
- Changing the seed (`4242`) — your checkpoints will never match the labs.
- Running in **PowerShell 7 vs Windows PowerShell 5.1**: both work, but `Get-Random` sequences differ between versions, so your exact rows may differ from a colleague's. Either is fine for the labs — totals are checked with tolerances, and structure is what matters.
- Saving the script with a `.txt` extension from Notepad ("Generate-NorthStarData.ps1.txt"). In the Save dialog choose *All files*.

**Troubleshooting:**

| Problem | Fix |
|---|---|
| `...cannot be loaded because running scripts is disabled` | You skipped the `Set-ExecutionPolicy -Scope Process ...` line; run it first (affects only the current window) |
| Script runs but folder empty | You probably edited `$dataDir`; restore the exact path |
| Takes forever (>10 min) | Antivirus scanning each write; exclude `C:\PowerBI_Labs` temporarily, or just wait |
| `Export-Csv : Access to the path ... denied` | A CSV is open in Excel — close Excel and rerun |

---

### Step 7 — Inspect the data (meet your defects)

**What:** Peek at the files so Lab_02's problems aren't a surprise:

```powershell
# First 5 sales rows
Import-Csv C:\PowerBI_Labs\NorthStar\data\sales.csv | Select-Object -First 5 | Format-Table

# Count the deliberate defects
$s = Import-Csv C:\PowerBI_Labs\NorthStar\data\sales.csv
'Null customer_ids : ' + ($s | Where-Object { -not $_.customer_id }).Count
'Negative qty rows : ' + ($s | Where-Object { [int]$_.quantity -lt 0 }).Count

# Region casing chaos in stores.csv
Import-Csv C:\PowerBI_Labs\NorthStar\data\stores.csv |
    Group-Object region | Sort-Object Name | Select-Object Name, Count
```

**Why:** Data profiling *before* loading into any tool is a professional reflex. You should never be surprised by your own data.

**Expected output:** ~1,000 null customer_ids (≈2%), ~500 negative-quantity rows (≈1%), and the region grouping showing ugly variants like `EAST`, `east`, `East` as *separate groups* — that's the Lab_02 cleaning list, previewed.

**Verify:** Counts are in those ballparks (exact values depend on PowerShell version — see Step 6 note).

**Common mistakes:** Panicking at the "bad" data. It's supposed to be there. **Do not fix the CSVs** — Lab_02 fixes them *inside Power Query*, which is the whole lesson (transformations should live in the tool, repeatably, not as one-off file edits).

**Troubleshooting:** `Import-Csv` on 50k rows takes ~10–20 seconds; that's normal.

---

### Step 8 — Final verification checkpoint

**What:** Run this consolidated check and keep the output:

```powershell
$checks = [ordered]@{
    'Power BI Desktop installed' = [bool](Get-StartApps | Where-Object Name -like '*Power BI Desktop*')
    'Folder structure exists'    = (Test-Path 'C:\PowerBI_Labs\NorthStar\data') -and
                                   (Test-Path 'C:\PowerBI_Labs\NorthStar\pbix')
    'stores.csv (85 rows)'       = (Import-Csv C:\PowerBI_Labs\NorthStar\data\stores.csv).Count -eq 85
    'products.csv (500 rows)'    = (Import-Csv C:\PowerBI_Labs\NorthStar\data\products.csv).Count -eq 500
    'customers.csv (2000 rows)'  = (Import-Csv C:\PowerBI_Labs\NorthStar\data\customers.csv).Count -eq 2000
    'promotions.csv (12 rows)'   = (Import-Csv C:\PowerBI_Labs\NorthStar\data\promotions.csv).Count -eq 12
    'sales.csv (50000 rows)'     = (Import-Csv C:\PowerBI_Labs\NorthStar\data\sales.csv).Count -eq 50000
    'targets.csv (96 rows)'      = (Import-Csv C:\PowerBI_Labs\NorthStar\data\targets.csv).Count -eq 96
}
$checks.GetEnumerator() | ForEach-Object {
    '{0} : {1}' -f ($_.Value ? 'PASS' : 'FAIL'), $_.Key
}
```

(The `? :` ternary needs PowerShell 7; on Windows PowerShell 5.1 replace the last block with `$checks.GetEnumerator() | ForEach-Object { if ($_.Value) {"PASS : $($_.Key)"} else {"FAIL : $($_.Key)"} }`.)

Also confirm manually, inside Power BI Desktop: **Options > Global > Data Load > Auto date/time is unchecked**, and note your version from **Help > About**.

**Why:** Every later lab assumes exactly this state. A five-minute checkpoint now saves an hour of confusion in Lab_02.

**Expected output:** Eight `PASS` lines.

**Verify:** All PASS + the two manual confirmations.

**Common mistakes:** Ignoring one FAIL "because it's probably fine." It isn't. Rerun the relevant step.

**Troubleshooting:** The Start-apps check can false-FAIL for the .exe install on some machines even when the app runs fine — if Power BI launches, count it as PASS.

---

**Lab_00 complete.** You have a configured tool and a realistic (deliberately imperfect) dataset. In [Lab_01_Power_BI_and_BI_Concepts.md](Lab_01_Power_BI_and_BI_Concepts.md) you'll load this data and build your first visual — and discover exactly why raw flat files aren't enough.

---

## 5. Production Engineering Practices

Setup habits are engineering habits. Five practices to adopt from day zero — each with a real-world failure story of the kind that makes these rules exist.

### 5.1 Standardize environments across the team

**What:** Document tool + version + settings (exactly what this lab is). Teams often keep a one-page "workstation setup" doc or a setup script in the repo.

**Why:** Power BI updates monthly; features and even DAX behavior details evolve. Mixed versions = "works on my machine" for BI.

**Failure story:** A consultancy delivered a .pbix built on the latest Desktop to a client whose IT had frozen Desktop at a 10-month-old version. The file wouldn't open at all — newer format. The team spent two days rebuilding features in the old version. A one-line "confirm client's Power BI version" checklist item would have prevented the entire incident.

### 5.2 Disable Auto date/time as team policy, not personal preference

**What:** The Step-4 setting, written into the team's standards doc.

**Why:** One developer with it enabled produces models that behave and size differently from everyone else's.

**Failure story:** A retail client's flagship model was 1.4 GB and refreshing at the edge of its capacity limits. An audit found 38 hidden auto date tables (the model had 19 datetime columns across many tables). Disabling the feature and pointing everything at one `dim_date` cut the file to under 600 MB overnight. Nobody had ever *chosen* those tables — they were a default nobody questioned.

### 5.3 Keep data, files, and scripts in predictable, non-synced paths

**What:** `C:\PowerBI_Labs\...` here; in production, agreed network/SharePoint conventions with .pbix files *edited locally* and never live-edited from a synced folder.

**Why:** OneDrive/Dropbox syncing a .pbix mid-save corrupts files; wandering data paths break refresh.

**Failure story:** An analyst kept the department's only copy of a critical .pbix in a OneDrive folder. A sync conflict during save produced a 0-byte file and a conflict copy that wouldn't open. Three weeks of work — measures, bookmarks, formatting — gone. (Versioned copies, next item, is the other half of the fix.)

### 5.4 Version everything — even "just a setup script"

**What:** The generator script belongs in version control (you learned git in SK-01). .pbix files are binary and diff poorly, so teams version them by disciplined file naming (`NorthStar_Lab01.pbix`, `NorthStar_Lab02.pbix`, dates or v-numbers in real projects) plus git for any extractable text (scripts, later: model metadata via Tabular Editor).

**Why:** "Which version of the generator produced this data?" must always be answerable, or your tests are built on sand.

**Failure story:** A team's synthetic test data script lived only on one engineer's laptop. He tweaked distributions casually for months; when he left, nobody could reproduce the test dataset the regression suite's expected values were based on, and 40+ "failing" tests had to be re-baselined by hand.

### 5.5 Seed your randomness; document your intentional defects

**What:** `Get-Random -SetSeed 4242`, and a header comment listing the injected data-quality issues.

**Why:** Reproducibility is the difference between *engineering* and *vibes*. And undocumented fake defects have a way of being mistaken for real ones.

**Failure story:** A QA team injected fake nulls into a staging dataset to test error handling — and didn't document it. Six months later a different team "discovered" the nulls, filed a data-quality incident against the upstream pipeline team, and two teams burned a week investigating a defect that was installed on purpose. The fix was a comment block.

---

## 6. Reflection

### What you learned

- How to install Power BI Desktop (Store vs direct download) and why monthly updates are the norm
- Where to check your version (**Help > About**) and why version awareness matters
- Why **Auto date/time** is disabled by professionals (bloat + wrong calendar for fiscal-year businesses like NorthStar)
- A clean, non-synced folder convention for BI work
- How to generate seeded, realistic synthetic data in PowerShell — including *deliberate* defects for later cleaning
- Basic data profiling in PowerShell before ever opening a BI tool

### Interview questions (with model answers)

**Q1. Power BI Desktop vs the Power BI Service — what's the difference?**
*Model answer:* Desktop is the free Windows authoring tool where you build queries, the data model, and reports, saved as .pbix. The Service is the cloud platform (app.powerbi.com) where reports are published, shared, refreshed on schedule, and secured for consumers. Authoring happens in Desktop; distribution and collaboration happen in the Service. (Lab_01 covers the full ecosystem.)

**Q2. What is Auto date/time and why would you disable it?**
*Model answer:* A default that creates a hidden calendar-based date table for every date column to enable drag-and-drop date hierarchies. I disable it because it bloats the model (often dramatically with many date columns), uses a Jan–Dec calendar that's wrong for fiscal-year businesses, and is redundant once you build a proper shared date dimension — which you need anyway for time intelligence.

**Q3. Why do professionals build one explicit date table instead of relying on hierarchies per date column?**
*Model answer:* A single marked date dimension gives one consistent calendar (including fiscal logic), lets all fact tables share the same date filters, enables DAX time-intelligence functions, and keeps model size down. Per-column hidden tables give none of that consistency.

**Q4. How would you create realistic test data when you can't use production data?**
*Model answer:* Generate synthetic data with a script: realistic distributions, referential integrity between dimension and fact keys, a fixed random seed for reproducibility, and deliberately injected quality issues to exercise cleaning logic — all documented and version-controlled. Production data is often off-limits for privacy/compliance reasons, so this is a standard practice.

**Q5. What's inside a .pbix file, at a high level?**
*Model answer:* It's a zip-like container holding the Power Query definitions (M code), the data model metadata (tables, relationships, DAX measures), the compressed imported data itself (VertiPaq), and the report layout definition. Lab_01 opens one up in detail.

**Q6. Your Power BI file works on your machine but a colleague can't open it. First suspicions?**
*Model answer:* Version mismatch — a .pbix saved in a newer Desktop version may not open in an older one, and Desktop updates monthly. Check both versions via Help > About and update the older install. Second suspicion: file corruption from a sync tool if it lived in OneDrive/Dropbox.

**Q7. Why seed a random data generator?**
*Model answer:* Determinism. The same seed produces the same sequence, so datasets are reproducible across runs and machines, expected values in checks stay valid, and you can distinguish "my logic changed" from "the data changed."

### Common interview traps

- **"Power BI is free, right?"** — *Desktop* is free. Sharing via the Service needs Pro/Premium licensing. Answering just "yes" or just "no" both lose points; distinguish authoring from distribution.
- **"Power BI runs on Mac?"** — No; Desktop is Windows-only. Mentioning the Service being browser-based (viewable anywhere) shows nuance.
- **Confusing Power Query with DAX.** Power Query/M transforms data *before* it lands in the model; DAX calculates *over* the loaded model. Interviewers probe this constantly; it becomes crisp in Labs 02 and 04.
- **Not knowing your tool's version cadence.** "It updates monthly, I check Help > About, teams standardize on staying current" is a small answer that signals real operational experience.

### Real-world applications

- Writing onboarding/setup documentation for a BI team (this lab is a template)
- Standing up reproducible test datasets for pipeline and report testing
- Auditing inherited models for Auto date/time bloat — one of the fastest "quick wins" a new hire can deliver
- Establishing file/folder/versioning conventions on a new analytics project

### Key takeaways

1. **Defaults are decisions someone else made for you.** Auto date/time is friendly for demos and hostile at scale — audit defaults.
2. **Reproducibility is a habit, not a feature.** Seeded data, versioned scripts, documented settings.
3. **Dirty data is normal.** You built the defects yourself this time; Lab_02 teaches you to fix them where fixes belong — inside the tool, repeatably.
4. **Environment discipline compounds.** Every later lab moves faster because this one was done carefully.

**Next up:** [Lab_01_Power_BI_and_BI_Concepts.md](Lab_01_Power_BI_and_BI_Concepts.md) — what BI actually is, the Power BI ecosystem, and your first (deliberately naive) report on the NorthStar data.
