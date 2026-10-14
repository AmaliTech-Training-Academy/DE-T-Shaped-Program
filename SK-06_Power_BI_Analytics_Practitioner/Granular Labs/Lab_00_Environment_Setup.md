# Lab 00 — Environment Setup

> **Goal:** Install and configure Power BI Desktop, run the NorthStar Retail source database (PostgreSQL 16 in Docker) filled with seeded data, and connect Power BI to it — the exact setup the guided lab starts from.
> **Time:** 2–3 hours | **Difficulty:** Beginner | **Prerequisites:** Windows 10/11 64-bit PC with admin rights, 8 GB RAM, ~10 GB free disk space
> **Guided lab coverage:** PART 0 of `SK-06_End_to_End_Guided_Lab.md` (Steps 0.1–0.5)

## Learning objectives

By the end of this lab you can:

- Explain what Power BI Desktop is made of (Power Query, the VertiPaq model, DAX, the report canvas) and how it differs from the Power BI Service.
- Install Power BI Desktop and switch off **Auto date/time** — the default that silently bloats models and uses the wrong calendar for NorthStar.
- Create the SK-06 project folder `C:\DataEngineering\sk06` and explain why it must not live in OneDrive.
- Start PostgreSQL 16 in Docker, generate the NorthStar data with a **seeded** Python script, and load it.
- Connect Power BI Desktop to the database and check the data against SQL **control totals**.

**Key terms**

| Term | Meaning |
|---|---|
| **Power BI Desktop** | Free Windows app where you build queries, the data model and the report. Saved as one `.pbix` file. |
| **Power BI Service** | The cloud portal (app.powerbi.com) where reports are published, shared, refreshed and secured. Covered in Lab 06. |
| **Power Query / M** | The data-preparation engine inside Power BI. Every click writes code in a language called **M** (Lab 02). |
| **VertiPaq** | The in-memory, **columnar** engine that stores imported data. A column with few distinct values (4 regions) compresses to almost nothing. |
| **DAX** | Data Analysis Expressions — the formula language for measures (Labs 04–05). |
| **Auto date/time** | A Power BI default that creates a hidden calendar table for every date column. We switch it off. |
| **OLTP source** | The operational database the business runs on. NorthStar's has 11 normalised tables. |
| **Seed** | A fixed starting value for a random-number generator. Same seed → same "random" data on every PC. |
| **Container / image / volume** | A running isolated program (`sk06-postgres`), the template it starts from (`postgres:16`), and the storage that survives restarts (`sk06_pgdata`). |

**Scenario.** NorthStar Retail has 85 stores in 4 regions, 12,000 SKUs (a SKU — stock-keeping unit — is one distinct sellable product) and 5 product categories. Every Monday the CFO gets a 40-page Excel report that three analysts compile over the weekend. Across Labs 01–06 you build its replacement: one Power BI dashboard. This lab builds your workshop and your data.

> **macOS / Linux:** Power BI Desktop is **Windows-only**. Use a Windows 11 virtual machine (Parallels, VMware Fusion, or a cloud Windows VM) with at least 8 GB RAM, and do every step inside it.

## Steps

### Step 1 — Check your system

Open **PowerShell** (press `Win`, type `powershell`, Enter — no admin needed) and run:

```powershell
Get-ComputerInfo -Property OsName, OsVersion, CsTotalPhysicalMemory | Format-List
"RAM (GB): " + [math]::Round((Get-ComputerInfo).CsTotalPhysicalMemory / 1GB, 1)
Get-PSDrive C | Select-Object @{n='FreeGB'; e={[math]::Round($_.Free / 1GB, 1)}}
```

**Why:** Power BI holds the whole data model in RAM, and Docker needs disk space for the database image.

**Expected output:** `OsName` shows Windows 10 or 11, `RAM (GB)` is about 8 or more, `FreeGB` is 10 or more.

✅ **Checkpoint:** Windows 64-bit, ≥ 8 GB RAM, ≥ 10 GB free on C:.

### Step 2 — Install Power BI Desktop

Choose ONE option:

| | **Option A: Microsoft Store (recommended)** | **Option B: Direct download** |
|---|---|---|
| How | Press `Win`, open **Microsoft Store**, search **Power BI Desktop** (publisher Microsoft Corporation), click **Get** | <https://www.microsoft.com/en-us/download/details.aspx?id=58494> → download `PBIDesktopSetup_x64.exe` → run it |
| Updates | Automatic, monthly | Manual — rerun a new installer each month |
| Admin rights | Not needed | Needed |

Power BI Desktop ships a new version **every month** and has no long-term-support version. A `.pbix` saved in a newer version may not open in an older one, so teams agree to stay current — the Store does that for you.

Launch Power BI Desktop. If it asks you to sign in, close the dialog — no account is needed until Lab 06.

**Expected output:** a blank report canvas.

**Check your version:** **Help** ribbon → **About**. Note the version and release month (for example "June 2026").

✅ **Checkpoint:** Power BI Desktop opens and **Help → About** shows a version from the last two months.

### Step 3 — Switch off Auto date/time

**File → Options and settings → Options**:

1. **GLOBAL → Data Load** → under *Time intelligence*, **uncheck "Auto date/time for new files"**.
2. **CURRENT FILE → Data Load** → uncheck **"Auto date/time"**.
3. Leave **Preview features** as they are. Click **OK**.

**Why this matters for NorthStar:**

- **Bloat.** Power BI would build a hidden calendar table for *every* date column (order date, ship date, registration date, promotion start/end …). In real models this is often a third or more of the file size.
- **Wrong calendar.** The hidden tables use January–December. NorthStar's **fiscal year starts 1 July** (FY2026 = 1 July 2025 – 30 June 2026). Every auto "Year" would be wrong for this business.
- **Broken habit.** Professionals build *one* explicit date table (`dim_date`, Lab 02) that every fact shares.

✅ **Checkpoint:** reopen Options → **GLOBAL → Data Load**: the box is still unchecked. (If it "comes back" in a new file, you only changed CURRENT FILE.)

### Step 4 — Create the project folder

```powershell
New-Item -ItemType Directory -Force C:\DataEngineering\sk06\data, C:\DataEngineering\sk06\scripts, C:\DataEngineering\sk06\sql, C:\DataEngineering\sk06\pbix | Out-Null
Set-Location C:\DataEngineering\sk06
Get-ChildItem
```

| Folder | Holds |
|---|---|
| `data` | the 11 generated CSV files |
| `scripts` | the Python generator |
| `sql` | SQL scripts (visible in the container as `/sql`) |
| `pbix` | your Power BI files, one per lab |

**Why not Documents or Desktop?** On most company laptops those are synced by **OneDrive**. OneDrive can corrupt a `.pbix` that is being saved while it syncs, and Docker often cannot share OneDrive folders with containers. Never put the project under a path containing `OneDrive`.

**Expected output:**

```
    Directory: C:\DataEngineering\sk06

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        ...                                  data
d-----        ...                                  pbix
d-----        ...                                  scripts
d-----        ...                                  sql
```

✅ **Checkpoint:** your prompt shows `PS C:\DataEngineering\sk06>` and the four folders exist.

### Step 5 — Install Docker Desktop

If Docker Desktop already works from SK-02 or SK-03, just start it and run the three verification commands.

1. Docker Desktop needs **WSL 2**. In an **admin** PowerShell run `wsl --install`, then reboot if asked.
2. Install Docker Desktop from <https://www.docker.com/products/docker-desktop/> or with `winget install -e --id Docker.DockerDesktop` (admin PowerShell).
3. Start **Docker Desktop** and wait for *Engine running*.

```powershell
docker --version
docker compose version
docker run --rm hello-world
```

**Expected output** (versions may be newer):

```
Docker version 27.x.x, build ...
Docker Compose version v2.x.x
...
Hello from Docker!
```

✅ **Checkpoint:** `hello-world` prints *Hello from Docker!*

### Step 6 — Write the Docker Compose file and start PostgreSQL

Create `C:\DataEngineering\sk06\docker-compose.yml` (VS Code or Notepad — in Notepad choose *Save as type: All files* so it does not become `.yml.txt`):

```yaml
services:
  postgres:
    image: postgres:16
    container_name: sk06-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres     # local learning container only — never reuse in real systems
      POSTGRES_DB: northstar
    ports:
      - "5434:5432"                   # Power BI connects to localhost:5434
    volumes:
      - sk06_pgdata:/var/lib/postgresql/data
      - ./data:/data:ro               # generated CSV files
      - ./sql:/sql:ro                 # SQL scripts

volumes:
  sk06_pgdata:
```

**What each part does:**

- `POSTGRES_DB: northstar` creates the `northstar` database on first start.
- `"5434:5432"` makes the database reachable from Windows as `localhost:5434`. Port 5434 avoids clashing with the SK-02 (5432) and SK-03 (5433) databases if those containers are still around.
- `./data:/data:ro` and `./sql:/sql:ro` let the database server read your CSV files and scripts (`ro` = read-only).

Start it:

```powershell
Set-Location C:\DataEngineering\sk06
docker compose up -d
docker exec sk06-postgres pg_isready -U postgres
```

The first run downloads the image (~400 MB).

**Expected output** (last line):

```
/var/run/postgresql:5432 - accepting connections
```

(Inside the container the port is still 5432; only Windows sees 5434.)

✅ **Checkpoint:** `pg_isready` says `accepting connections`.

### Step 7 — Generate the NorthStar data with Python

Check Python (3.12 recommended, 3.11+ works):

```powershell
python --version
```

The generator uses only the Python standard library, so no packages and no virtual environment are needed. Save this as `C:\DataEngineering\sk06\scripts\generate_northstar.py` — it is identical to Step 0.2 of the guided lab:

```python
"""Generate the NorthStar Retail source data for SK-06 (Power BI).

Writes 11 CSV files into the folder given by --out (default: data).
Only the Python standard library is used. The random seed is fixed, so
every trainee gets exactly the same rows and the same lab numbers.
"""
import argparse
import bisect
import csv
import datetime as dt
import random
from pathlib import Path

SEED = 20260701
FIRST_DAY = dt.date(2023, 7, 1)   # start of FY2024
LAST_DAY = dt.date(2026, 6, 30)   # end of FY2026

REGIONS = ["Northeast", "Southeast", "Midwest", "West"]
# region -> list of (district, [(city, state, lat, lon), ...])
GEO = {
    "Northeast": [
        ("New England", [("Boston", "MA", 42.36, -71.06), ("Hartford", "CT", 41.77, -72.67), ("Providence", "RI", 41.82, -71.41)]),
        ("New York Metro", [("New York", "NY", 40.71, -74.01), ("Newark", "NJ", 40.74, -74.17), ("Albany", "NY", 42.65, -73.76)]),
        ("Pennsylvania", [("Philadelphia", "PA", 39.95, -75.17), ("Pittsburgh", "PA", 40.44, -79.99)]),
    ],
    "Southeast": [
        ("Georgia", [("Atlanta", "GA", 33.75, -84.39), ("Savannah", "GA", 32.08, -81.09)]),
        ("Florida", [("Miami", "FL", 25.76, -80.19), ("Orlando", "FL", 28.54, -81.38), ("Tampa", "FL", 27.95, -82.46)]),
        ("Carolinas & Tennessee", [("Charlotte", "NC", 35.23, -80.84), ("Raleigh", "NC", 35.78, -78.64), ("Nashville", "TN", 36.16, -86.78)]),
    ],
    "Midwest": [
        ("Great Lakes", [("Chicago", "IL", 41.88, -87.63), ("Milwaukee", "WI", 43.04, -87.91), ("Detroit", "MI", 42.33, -83.05)]),
        ("Ohio Valley", [("Columbus", "OH", 39.96, -83.00), ("Indianapolis", "IN", 39.77, -86.16)]),
        ("Plains", [("Minneapolis", "MN", 44.98, -93.27), ("St. Louis", "MO", 38.63, -90.20), ("Kansas City", "MO", 39.10, -94.58)]),
    ],
    "West": [
        ("Pacific Northwest", [("Seattle", "WA", 47.61, -122.33), ("Portland", "OR", 45.52, -122.68)]),
        ("California", [("Los Angeles", "CA", 34.05, -118.24), ("San Diego", "CA", 32.72, -117.16), ("San Jose", "CA", 37.34, -121.89)]),
        ("Mountain", [("Denver", "CO", 39.74, -104.99), ("Phoenix", "AZ", 33.45, -112.07), ("Salt Lake City", "UT", 40.76, -111.89)]),
    ],
}
STORES_PER_REGION = {"Northeast": 24, "Southeast": 22, "Midwest": 21, "West": 18}  # 85 stores

CATALOG = {  # category -> [(subcategory, min_price, max_price)]
    "Apparel": [("Tops", 12, 60), ("Bottoms", 20, 90), ("Outerwear", 45, 250), ("Footwear", 30, 180)],
    "Electronics": [("Audio", 20, 400), ("Computing", 80, 1200), ("Mobile Accessories", 8, 80), ("Smart Home", 25, 300)],
    "Home & Garden": [("Kitchen", 10, 200), ("Furniture", 60, 900), ("Decor", 8, 120), ("Garden", 10, 250)],
    "Grocery": [("Snacks", 2, 12), ("Beverages", 2, 25), ("Pantry", 3, 30), ("Household", 3, 40)],
    "Sports & Outdoors": [("Fitness", 10, 400), ("Camping", 15, 350), ("Cycling", 20, 900), ("Team Sports", 8, 150)],
}
CATEGORY_WEIGHT = {"Apparel": 1.2, "Electronics": 0.7, "Home & Garden": 0.9, "Grocery": 2.0, "Sports & Outdoors": 0.8}
BRANDS = ["Northwind", "Apex", "Lumen", "Harbor", "Summit", "Vista", "Crescent", "Evergreen"]
PRODUCTS_PER_SUBCATEGORY = 600  # 5 x 4 x 600 = 12,000 SKUs

FIRST_NAMES = ["Ama", "Kwame", "Efua", "Kofi", "Sarah", "James", "Linda", "David", "Grace", "Michael",
               "Akosua", "Yaw", "Esi", "Daniel", "Rita", "Paul", "Naomi", "Victor", "Abena", "Samuel"]
LAST_NAMES = ["Mensah", "Owusu", "Johnson", "Smith", "Boateng", "Brown", "Asante", "Williams", "Osei", "Davis",
              "Appiah", "Miller", "Addo", "Wilson", "Agyeman", "Taylor", "Ansah", "Clark", "Frimpong", "Lewis"]
SEGMENTS = (["Value", "Regular", "Premium"], [50, 35, 15])

MONTH_FACTOR = {1: 0.85, 2: 0.85, 3: 0.95, 4: 0.95, 5: 1.0, 6: 1.0, 7: 1.0, 8: 1.05, 9: 0.95, 10: 1.0, 11: 1.25, 12: 1.45}
WEEKDAY_FACTOR = {0: 0.92, 1: 0.92, 2: 0.92, 3: 0.95, 4: 1.05, 5: 1.30, 6: 1.15}  # Mon..Sun
FY_GROWTH = {2024: 1.00, 2025: 1.07, 2026: 1.15}
BASE_ORDERS_PER_DAY = 140


def fiscal_year(d):
    return d.year + 1 if d.month >= 7 else d.year


def write_csv(folder, name, header, rows):
    path = folder / name
    with path.open("w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(header)
        w.writerows(rows)
    print(f"  {name:<22} {len(rows):>8,} rows")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--out", default="data")
    out = Path(parser.parse_args().out)
    out.mkdir(parents=True, exist_ok=True)
    rng = random.Random(SEED)
    print(f"Generating NorthStar Retail data (seed {SEED}) into {out.resolve()}")

    # ---- geography: regions, districts, stores --------------------------
    regions = [(i + 1, r) for i, r in enumerate(REGIONS)]
    districts, stores = [], []
    store_weight = []
    district_id = 0
    store_id = 100
    for region_id, region in regions:
        geo = GEO[region]
        region_districts = []
        for district_name, cities in geo:
            district_id += 1
            districts.append((district_id, district_name, region_id))
            region_districts.append((district_id, cities))
        for n in range(STORES_PER_REGION[region]):
            d_id, cities = region_districts[n % len(region_districts)]
            city, state, lat, lon = cities[(n // len(region_districts)) % len(cities)]
            store_id += 1
            open_date = dt.date(2008, 1, 1) + dt.timedelta(days=rng.randint(0, 5000))
            stores.append((store_id, f"NorthStar {city} #{store_id}", d_id, city, state,
                           round(lat + rng.uniform(-0.15, 0.15), 6), round(lon + rng.uniform(-0.15, 0.15), 6),
                           open_date.isoformat()))
            store_weight.append(rng.uniform(0.6, 1.6))
    store_ids = [s[0] for s in stores]
    store_cum = list(_cumulative(store_weight))

    # ---- products ----------------------------------------------------------
    categories, subcategories, products = [], [], []
    product_weight = []
    product_price, product_cost = {}, {}
    sub_id = 0
    product_id = 10000
    for cat_id, (cat, subs) in enumerate(CATALOG.items(), start=1):
        categories.append((cat_id, cat))
        for sub, lo, hi in subs:
            sub_id += 1
            subcategories.append((sub_id, sub, cat_id))
            for _ in range(PRODUCTS_PER_SUBCATEGORY):
                product_id += 1
                brand = rng.choice(BRANDS)
                price = round(rng.uniform(lo, hi), 2)
                cost = round(price * rng.uniform(0.55, 0.72), 2)
                products.append((product_id, f"{brand} {sub} {product_id}", sub_id, brand,
                                 f"{cost:.2f}", f"{price:.2f}"))
                product_price[product_id] = price
                product_cost[product_id] = cost
                # long-tail popularity: a few products sell a lot (Pareto)
                popularity = 1.0 / (rng.randint(1, PRODUCTS_PER_SUBCATEGORY) ** 0.9)
                product_weight.append(popularity * CATEGORY_WEIGHT[cat] / (price ** 0.35))
    product_ids = [p[0] for p in products]
    product_cum = list(_cumulative(product_weight))

    # ---- customers ---------------------------------------------------------
    customers = []
    n_customers = 30000
    span_before = (FIRST_DAY - dt.date(2019, 1, 1)).days
    span_during = (LAST_DAY - FIRST_DAY).days
    reg_dates = []
    for _ in range(n_customers):
        if rng.random() < 0.40:
            reg = dt.date(2019, 1, 1) + dt.timedelta(days=rng.randint(0, span_before - 1))
        else:
            reg = FIRST_DAY + dt.timedelta(days=rng.randint(0, span_during))
        reg_dates.append(reg)
    reg_dates.sort()
    for i, reg in enumerate(reg_dates, start=1):
        first, last = rng.choice(FIRST_NAMES), rng.choice(LAST_NAMES)
        city, state = rng.choice([(c[0], c[1]) for g in GEO.values() for _, cs in g for c in cs])
        segment = rng.choices(*SEGMENTS)[0]
        customers.append((i, first, last, f"customer{i}@example.com", segment, reg.isoformat(), city, state))
    customer_by_reg_day = {}
    for c in customers:
        customer_by_reg_day.setdefault(c[5], []).append(c[0])

    # ---- promotions ----------------------------------------------------------
    promotions = [(0, "No Promotion", "None", "0.00", FIRST_DAY.isoformat(), LAST_DAY.isoformat())]
    promo_templates = [("Back to School", "Seasonal", 8, 10), ("Black Friday", "Holiday", 11, 24),
                       ("Holiday Gift Event", "Holiday", 12, 10), ("New Year Clearance", "Clearance", 1, 3),
                       ("Spring Refresh", "Seasonal", 3, 15), ("Summer Loyalty Days", "Loyalty", 6, 5)]
    promo_days = {}
    pid = 0
    for fy in (2024, 2025, 2026):
        for name, ptype, month, day in promo_templates:
            pid += 1
            year = fy - 1 if month >= 7 else fy
            start = dt.date(year, month, day)
            end = start + dt.timedelta(days=rng.randint(6, 16))
            pct = rng.choice([0.10, 0.15, 0.20, 0.25])
            promotions.append((pid, f"{name} {year}", ptype, f"{pct:.2f}", start.isoformat(), end.isoformat()))
            d = start
            while d <= end:
                promo_days[d] = (pid, pct)
                d += dt.timedelta(days=1)

    # ---- orders and order lines ------------------------------------------------
    orders, lines = [], []
    order_id = 1000000
    line_id = 0
    store_month_revenue = {}
    day = FIRST_DAY
    total_days = (LAST_DAY - FIRST_DAY).days
    while day <= LAST_DAY:
        fy = fiscal_year(day)
        trend = 1 + 0.03 * ((day - FIRST_DAY).days / total_days)
        expected = BASE_ORDERS_PER_DAY * MONTH_FACTOR[day.month] * WEEKDAY_FACTOR[day.weekday()] * FY_GROWTH[fy] * trend
        n_orders = max(1, int(round(rng.gauss(expected, expected ** 0.5))))
        bound = bisect.bisect_right(reg_dates, day)  # customers registered on/before this day
        new_today = customer_by_reg_day.get(day.isoformat(), [])
        promo = promo_days.get(day)
        for k in range(n_orders):
            order_id += 1
            # every customer who registers today places their first order today
            customer_id = new_today[k] if k < len(new_today) else rng.randint(1, bound)
            r = rng.random()
            ship_date = "" if r < 0.01 else (day + dt.timedelta(days=rng.choices([1, 2, 3, 4, 5, 6, 7], [10, 25, 25, 15, 10, 8, 7])[0])).isoformat()
            is_returned = 1 if rng.random() < 0.04 else 0
            orders.append((order_id, day.isoformat(), ship_date, customer_id, is_returned, "pos_loader", "POS"))
            store = store_ids[bisect.bisect_left(store_cum, rng.random() * store_cum[-1])]
            n_lines = rng.choices([1, 2, 3, 4], [50, 30, 15, 5])[0]
            for line_number in range(1, n_lines + 1):
                line_id += 1
                prod = product_ids[bisect.bisect_left(product_cum, rng.random() * product_cum[-1])]
                qty = rng.choices([1, 2, 3, 4], [60, 25, 10, 5])[0]
                price = product_price[prod]
                promo_id, discount = 0, 0.0
                if promo and rng.random() < 0.35:
                    promo_id = promo[0]
                    discount = round(qty * price * promo[1], 2)
                lines.append((line_id, order_id, line_number, prod, store, promo_id, qty, f"{price:.2f}", f"{discount:.2f}"))
                key = (store, day.replace(day=1))
                store_month_revenue[key] = store_month_revenue.get(key, 0.0) + qty * price - discount
        day += dt.timedelta(days=1)

    # ---- monthly revenue targets per store ---------------------------------------
    # targets were set ambitiously for some regions, so the RAG status varies
    region_of_store = {s[0]: REGIONS[districts[s[2] - 1][2] - 1] for s in stores}
    ambition = {"Northeast": 0.97, "Southeast": 1.03, "Midwest": 1.07, "West": 1.14}
    targets = []
    for (store, month), revenue in sorted(store_month_revenue.items()):
        factor = ambition[region_of_store[store]] * rng.uniform(0.95, 1.05)
        target = round(revenue * factor, -2)
        targets.append((store, month.isoformat(), f"{target:.2f}"))

    # ---- write files ---------------------------------------------------------------
    write_csv(out, "regions.csv", ["region_id", "region_name"], regions)
    write_csv(out, "districts.csv", ["district_id", "district_name", "region_id"], districts)
    write_csv(out, "stores.csv", ["store_id", "store_name", "district_id", "city", "state", "latitude", "longitude", "open_date"], stores)
    write_csv(out, "categories.csv", ["category_id", "category_name"], categories)
    write_csv(out, "subcategories.csv", ["subcategory_id", "subcategory_name", "category_id"], subcategories)
    write_csv(out, "products.csv", ["product_id", "product_name", "subcategory_id", "brand", "unit_cost", "list_price"], products)
    write_csv(out, "customers.csv", ["customer_id", "first_name", "last_name", "email", "customer_segment", "registration_date", "city", "state"], customers)
    write_csv(out, "promotions.csv", ["promotion_id", "promotion_name", "promotion_type", "discount_pct", "start_date", "end_date"], promotions)
    write_csv(out, "orders.csv", ["order_id", "order_date", "ship_date", "customer_id", "is_returned", "created_by", "source_system"], orders)
    write_csv(out, "order_lines.csv", ["order_line_id", "order_id", "line_number", "product_id", "store_id", "promotion_id", "quantity", "unit_price", "discount_amount"], lines)
    write_csv(out, "revenue_targets.csv", ["store_id", "target_month", "revenue_target"], targets)
    print("Done.")


def _cumulative(weights):
    total = 0.0
    for w in weights:
        total += w
        yield total


if __name__ == "__main__":
    main()
```

**How the script works (skim it once):**

- `random.Random(SEED)` — every random choice comes from one generator with a fixed seed, so your rows are identical to everyone else's and to the numbers in these labs. **Do not change the seed.**
- **Geography:** 4 regions → 12 districts → 85 stores (ids 101–185; Northeast 101–124, Southeast 125–146, Midwest 147–167, West 168–185).
- **Products:** 5 categories × 4 subcategories × 600 = 12,000 SKUs, with a "long tail" — a few products sell a lot (you will measure the 80/20 rule in Lab 05).
- **Orders:** three full fiscal years, FY2024–FY2026 (1 July 2023 – 30 June 2026), with weekend and November/December peaks and ~7–10% yearly growth. About 1% of orders never ship (`ship_date` empty) and ~4% are returned.
- **Targets:** one revenue target per store per month, set more ambitiously in some regions so the target traffic lights in Lab 05 show green, amber and red.
- `orders.csv` also contains two columns the business does not need for reporting (`created_by`, `source_system`) — real sources always do. You will leave them behind in Lab 02.

Run it from the project folder:

```powershell
Set-Location C:\DataEngineering\sk06
python scripts\generate_northstar.py --out data
```

**Expected output** (about 15 seconds):

```
Generating NorthStar Retail data (seed 20260701) into C:\DataEngineering\sk06\data
  regions.csv                   4 rows
  districts.csv                12 rows
  stores.csv                   85 rows
  categories.csv                5 rows
  subcategories.csv            20 rows
  products.csv             12,000 rows
  customers.csv            30,000 rows
  promotions.csv               19 rows
  orders.csv              177,070 rows
  order_lines.csv         310,024 rows
  revenue_targets.csv       3,060 rows
Done.
```

Peek at the two biggest files:

```powershell
Get-Content data\orders.csv -TotalCount 3
Get-Content data\order_lines.csv -TotalCount 3
```

**Expected output:**

```
order_id,order_date,ship_date,customer_id,is_returned,created_by,source_system
1000001,2023-07-01,2023-07-03,12000,0,pos_loader,POS
1000002,2023-07-01,2023-07-03,12001,0,pos_loader,POS
order_line_id,order_id,line_number,product_id,store_id,promotion_id,quantity,unit_price,discount_amount
1,1000001,1,18707,171,0,2,28.22,0.00
2,1000002,1,18982,103,0,1,23.99,0.00
```

Notice there is **no revenue column** and **no region name** on a sales row — only ids. That is normal for an OLTP database, and it is why Lab 02 reshapes the data.

✅ **Checkpoint:** 11 CSV files in `C:\DataEngineering\sk06\data` and the row counts above match exactly.

### Step 8 — Create and load the source database

Save this as `C:\DataEngineering\sk06\sql\01_create_and_load_northstar.sql` (identical to Step 0.3 of the guided lab):

```sql
-- NorthStar Retail source database (OLTP-style) for SK-06
-- Run inside the container:  psql -U postgres -d northstar -f /sql/01_create_and_load_northstar.sql
-- Safe to re-run: it drops and recreates the three schemas.
\set ON_ERROR_STOP on

DROP SCHEMA IF EXISTS sales CASCADE;
DROP SCHEMA IF EXISTS products CASCADE;
DROP SCHEMA IF EXISTS finance CASCADE;
CREATE SCHEMA sales;
CREATE SCHEMA products;
CREATE SCHEMA finance;

-- ---------- geography ----------
CREATE TABLE sales.regions (
    region_id    INT PRIMARY KEY,
    region_name  TEXT NOT NULL UNIQUE
);
CREATE TABLE sales.districts (
    district_id    INT PRIMARY KEY,
    district_name  TEXT NOT NULL,
    region_id      INT NOT NULL REFERENCES sales.regions(region_id)
);
CREATE TABLE sales.stores (
    store_id     INT PRIMARY KEY,
    store_name   TEXT NOT NULL,
    district_id  INT NOT NULL REFERENCES sales.districts(district_id),
    city         TEXT NOT NULL,
    state        CHAR(2) NOT NULL,
    latitude     NUMERIC(9,6),
    longitude    NUMERIC(9,6),
    open_date    DATE
);

-- ---------- product hierarchy ----------
CREATE TABLE products.categories (
    category_id    INT PRIMARY KEY,
    category_name  TEXT NOT NULL UNIQUE
);
CREATE TABLE products.subcategories (
    subcategory_id    INT PRIMARY KEY,
    subcategory_name  TEXT NOT NULL,
    category_id       INT NOT NULL REFERENCES products.categories(category_id)
);
CREATE TABLE products.products (
    product_id      INT PRIMARY KEY,
    product_name    TEXT NOT NULL,
    subcategory_id  INT NOT NULL REFERENCES products.subcategories(subcategory_id),
    brand           TEXT NOT NULL,
    unit_cost       NUMERIC(10,2) NOT NULL,
    list_price      NUMERIC(10,2) NOT NULL
);

-- ---------- customers and promotions ----------
CREATE TABLE sales.customers (
    customer_id        INT PRIMARY KEY,
    first_name         TEXT NOT NULL,
    last_name          TEXT NOT NULL,
    email              TEXT NOT NULL,
    customer_segment   TEXT NOT NULL,
    registration_date  DATE NOT NULL,
    city               TEXT,
    state              CHAR(2)
);
CREATE TABLE sales.promotions (
    promotion_id    INT PRIMARY KEY,
    promotion_name  TEXT NOT NULL,
    promotion_type  TEXT NOT NULL,
    discount_pct    NUMERIC(4,2) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL
);

-- ---------- transactions ----------
CREATE TABLE sales.orders (
    order_id       BIGINT PRIMARY KEY,
    order_date     DATE NOT NULL,
    ship_date      DATE,                 -- NULL = not shipped (cancelled)
    customer_id    INT NOT NULL REFERENCES sales.customers(customer_id),
    is_returned    SMALLINT NOT NULL,    -- 1 = order was returned
    created_by     TEXT,                 -- audit column (not needed for BI)
    source_system  TEXT                  -- pipeline column (not needed for BI)
);
CREATE TABLE sales.order_lines (
    order_line_id    BIGINT PRIMARY KEY,
    order_id         BIGINT NOT NULL REFERENCES sales.orders(order_id),
    line_number      INT NOT NULL,
    product_id       INT NOT NULL REFERENCES products.products(product_id),
    store_id         INT NOT NULL REFERENCES sales.stores(store_id),
    promotion_id     INT NOT NULL REFERENCES sales.promotions(promotion_id),
    quantity         INT NOT NULL,
    unit_price       NUMERIC(10,2) NOT NULL,
    discount_amount  NUMERIC(10,2) NOT NULL
);

-- ---------- monthly targets (one row per store per month) ----------
CREATE TABLE finance.revenue_targets (
    store_id        INT NOT NULL REFERENCES sales.stores(store_id),
    target_month    DATE NOT NULL,       -- always the 1st of the month
    revenue_target  NUMERIC(12,2) NOT NULL,
    PRIMARY KEY (store_id, target_month)
);

-- ---------- load the CSV files (mounted at /data in the container) ----------
COPY sales.regions            FROM '/data/regions.csv'         WITH (FORMAT csv, HEADER true);
COPY sales.districts          FROM '/data/districts.csv'       WITH (FORMAT csv, HEADER true);
COPY sales.stores             FROM '/data/stores.csv'          WITH (FORMAT csv, HEADER true);
COPY products.categories      FROM '/data/categories.csv'      WITH (FORMAT csv, HEADER true);
COPY products.subcategories   FROM '/data/subcategories.csv'   WITH (FORMAT csv, HEADER true);
COPY products.products        FROM '/data/products.csv'        WITH (FORMAT csv, HEADER true);
COPY sales.customers          FROM '/data/customers.csv'       WITH (FORMAT csv, HEADER true);
COPY sales.promotions         FROM '/data/promotions.csv'      WITH (FORMAT csv, HEADER true);
COPY sales.orders             FROM '/data/orders.csv'          WITH (FORMAT csv, HEADER true);
COPY sales.order_lines        FROM '/data/order_lines.csv'     WITH (FORMAT csv, HEADER true);
COPY finance.revenue_targets  FROM '/data/revenue_targets.csv' WITH (FORMAT csv, HEADER true);

CREATE INDEX ix_order_lines_order ON sales.order_lines(order_id);
CREATE INDEX ix_orders_order_date ON sales.orders(order_date);
ANALYZE;

-- ---------- row-count check ----------
SELECT 'sales.regions' AS table_name, COUNT(*) AS row_count FROM sales.regions
UNION ALL SELECT 'sales.districts',          COUNT(*) FROM sales.districts
UNION ALL SELECT 'sales.stores',             COUNT(*) FROM sales.stores
UNION ALL SELECT 'products.categories',      COUNT(*) FROM products.categories
UNION ALL SELECT 'products.subcategories',   COUNT(*) FROM products.subcategories
UNION ALL SELECT 'products.products',        COUNT(*) FROM products.products
UNION ALL SELECT 'sales.customers',          COUNT(*) FROM sales.customers
UNION ALL SELECT 'sales.promotions',         COUNT(*) FROM sales.promotions
UNION ALL SELECT 'sales.orders',             COUNT(*) FROM sales.orders
UNION ALL SELECT 'sales.order_lines',        COUNT(*) FROM sales.order_lines
UNION ALL SELECT 'finance.revenue_targets',  COUNT(*) FROM finance.revenue_targets;
```

**What it does:** three **schemas** (named folders inside the database: `sales`, `products`, `finance`) and 11 tables with primary and foreign keys — a typical normalised OLTP design. `COPY ... FROM '/data/...'` is run by the database server itself, which reads the CSVs through the `./data:/data` mount. In CSV format an empty field is loaded as `NULL`, so unshipped orders get a `NULL` ship date.

Run it:

```powershell
docker exec sk06-postgres psql -U postgres -d northstar -f /sql/01_create_and_load_northstar.sql
```

**Expected output** (the first run prints NOTICEs that the schemas do not exist yet — that is fine; the end of the output is):

```
       table_name        | row_count
-------------------------+-----------
 sales.regions           |         4
 sales.districts         |        12
 sales.stores            |        85
 products.categories     |         5
 products.subcategories  |        20
 products.products       |     12000
 sales.customers         |     30000
 sales.promotions        |        19
 sales.orders            |    177070
 sales.order_lines       |    310024
 finance.revenue_targets |      3060
(11 rows)
```

✅ **Checkpoint:** all 11 row counts match. The script is safe to re-run — it drops and recreates the schemas.

### Step 9 — Record the control totals

A **control total** is a number computed *outside* Power BI that your report must reproduce. Open psql:

```powershell
docker exec -it sk06-postgres psql -U postgres -d northstar
```

At the `northstar=#` prompt run (the semicolon matters):

```sql
SELECT
    CASE WHEN EXTRACT(MONTH FROM o.order_date) >= 7
         THEN EXTRACT(YEAR FROM o.order_date) + 1
         ELSE EXTRACT(YEAR FROM o.order_date) END                     AS fiscal_year,
    ROUND(SUM(ol.quantity * ol.unit_price - ol.discount_amount), 2)   AS net_revenue,
    COUNT(DISTINCT o.order_id)                                         AS orders
FROM sales.orders o
JOIN sales.order_lines ol ON ol.order_id = o.order_id
GROUP BY 1
ORDER BY 1;
```

**Expected output:**

```
 fiscal_year | net_revenue | orders
-------------+-------------+--------
        2024 |  9958772.62 |  54446
        2025 | 10783033.72 |  58745
        2026 | 11862350.26 |  63879
(3 rows)
```

The `CASE` puts July–December into the *next* fiscal year — NorthStar names a fiscal year after the year it ends in. **Net revenue** = quantity × unit price − discount, the CFO's definition. Copy these three lines into a notes file; Labs 04 and 05 check your measures against them. Quit with `\q`.

✅ **Checkpoint:** FY2026 net revenue is **11,862,350.26** with **63,879** orders.

### Step 10 — Connect Power BI Desktop to PostgreSQL (smoke test)

1. In Power BI Desktop: **Home → Get data → More… → Database → PostgreSQL database → Connect**.
2. **Server:** `localhost:5434` **Database:** `northstar` · Data Connectivity mode **Import** → **OK**.
3. Credentials: choose the **Database** tab (not Windows), user `postgres`, password `postgres` → **Connect**.
4. If a dialog says it cannot connect using an encrypted connection, click **OK** to connect without encryption. The local container has no TLS certificate; a production server would have one.
5. In the Navigator, expand `northstar` and tick `sales.regions` → **Load**.

**Expected output:** the Data pane shows `sales regions` (Power BI turns the dot into a space); Table view shows 4 rows: Northeast, Southeast, Midwest, West.

Close Power BI **without saving** — this was only a connection test. Lab 01 starts the first real file.

✅ **Checkpoint:** Power BI shows the 4 regions from your container.

### Step 11 — Final verification

```powershell
Set-Location C:\DataEngineering\sk06
$checks = [ordered]@{
    'Power BI Desktop installed'      = [bool](Get-StartApps | Where-Object Name -like '*Power BI Desktop*')
    'Project folders exist'           = (Test-Path .\data) -and (Test-Path .\scripts) -and (Test-Path .\sql) -and (Test-Path .\pbix)
    '11 CSV files generated'          = ((Get-ChildItem .\data\*.csv).Count -eq 11)
    'Container sk06-postgres running' = ((docker inspect -f '{{.State.Running}}' sk06-postgres) -eq 'true')
    'order_lines loaded (310024)'     = ((docker exec sk06-postgres psql -U postgres -d northstar -tAc "SELECT COUNT(*) FROM sales.order_lines") -eq '310024')
}
foreach ($c in $checks.GetEnumerator()) {
    if ($c.Value) { "PASS : $($c.Key)" } else { "FAIL : $($c.Key)" }
}
```

**Expected output:**

```
PASS : Power BI Desktop installed
PASS : Project folders exist
PASS : 11 CSV files generated
PASS : Container sk06-postgres running
PASS : order_lines loaded (310024)
```

Also confirm by hand: **Options → GLOBAL → Data Load → Auto date/time** is unchecked.

✅ **Checkpoint:** five PASS lines plus Auto date/time off. (The first check can wrongly FAIL for the `.exe` install on some machines — if Power BI starts, count it as PASS.)

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Store **Get** button greyed out | Microsoft Store blocked by IT | Use Option B (direct download) or ask IT to deploy Power BI Desktop. |
| Power BI starts and closes immediately, or the canvas is blank | Graphics driver / VM without 3D acceleration | Update the GPU driver; in a VM enable 3D acceleration; or Options → GLOBAL → Diagnostics/Report settings → use software rendering, then restart. |
| WebView2 error on install | Edge WebView2 runtime missing | Install "Microsoft Edge WebView2 Runtime", then reinstall Power BI Desktop. |
| `docker: error during connect` | Docker Desktop not running | Start Docker Desktop, wait for *Engine running*. |
| `Bind for 0.0.0.0:5434 failed: port is already allocated` | Something else uses 5434 | `docker ps` to find it, or change the mapping to `"5435:5432"` and use `localhost:5435` in Power BI. |
| `python` opens the Microsoft Store or is not found | Python not installed / Store alias | Install Python 3.12 from python.org (tick *Add python.exe to PATH*) or `winget install -e --id Python.Python.3.12`, then open a new PowerShell. |
| Row counts differ from the lab | Script edited, seed changed, or saved from a partial copy | Re-save the script exactly, rerun Step 7, then rerun Step 8. |
| `could not open file "/data/regions.csv"` | Container started before the files existed, or project under OneDrive | Check `Get-ChildItem C:\DataEngineering\sk06\data`; then `docker compose down` and `docker compose up -d` from `C:\DataEngineering\sk06`. |
| `/sql/01_create_and_load_northstar.sql: No such file or directory` | File saved elsewhere or as `.sql.txt` | `Get-ChildItem C:\DataEngineering\sk06\sql` and fix the name. |
| Power BI: *password authentication failed* | Wrong credential type or password | Data source settings → `localhost:5434;northstar` → Edit Permissions → Edit → **Database** tab, `postgres` / `postgres`. |
| Power BI: *An error happened while reading data from the provider* / *Npgsql* message | Very old Power BI Desktop | Update Power BI Desktop (current versions include the PostgreSQL driver). |
| Power BI cannot reach `localhost:5434` | Container stopped | `docker compose start` from `C:\DataEngineering\sk06`, then refresh. |

## Teardown

Between labs, stop the database to free memory (data is kept):

```powershell
Set-Location C:\DataEngineering\sk06
docker compose stop      # stop, keep data
docker compose start     # start again before the next lab
```

To start completely clean (deletes the database; rerun Step 8 afterwards):

```powershell
docker compose down -v
docker compose up -d
```

## Key takeaways

- Power BI Desktop = Power Query (prepare) + VertiPaq model (store) + DAX (calculate) + report canvas (show), all in one `.pbix`. The Service is where it is shared — Lab 06.
- **Auto date/time is off** — NorthStar's fiscal year starts 1 July and one explicit `dim_date` will serve every fact.
- The source is a normalised OLTP database: PostgreSQL 16 in container `sk06-postgres`, database `northstar`, Windows port **5434**, user/password `postgres`/`postgres`.
- The data is seeded: everyone has the same 310,024 order lines, and FY2026 net revenue is **11,862,350.26**. Control totals computed in SQL are your answer key for every later lab.
- Keep the project in `C:\DataEngineering\sk06`, never in OneDrive.

**Interview check:** *Why seed a data generator?* — Determinism: the same seed gives the same data on every run and every machine, so expected values stay valid and you can tell "my logic changed" from "the data changed". *Desktop vs Service?* — Desktop is the free Windows authoring tool; the Service is the cloud platform for publishing, refresh, sharing and security.

## Next

[Lab 01 — Power BI & BI Concepts](Lab_01_Power_BI_and_BI_Concepts.md): build a first, deliberately naive report straight on the OLTP tables — and see exactly why the guided lab needs a star schema.
