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
| **Why a star schema instead of keeping the source OLTP tables?** | The source database has 11 tables joined by 10 foreign-key relationships. Every visual would require Power BI to walk all those joins. A star schema flattens dimensions so each visual needs one hop from a dimension to a fact — far faster queries and a model business users can understand. |
| **Why build the date dimension in Power Query rather than importing from SQL?** | The source database has no date table at all, and NorthStar's fiscal calendar (July 1 start) exists nowhere in it. Building it in M means the fiscal logic is in the Power BI model, visible and maintainable by the BI team — not buried in a database object owned by a different team. |
| **Why use measures instead of calculated columns for revenue metrics?** | Calculated columns are computed at refresh time and stored in the model (increasing its size). Measures compute dynamically in the current filter context — this is what allows one formula to show $11.9M for the whole company in FY2026 and $3.3M for the Northeast region. |
| **Why RLS rather than separate reports per region?** | Five separate reports (one per region + one company) means five times the maintenance work for every change. One report with RLS means one change propagates to all users automatically. |

---

## WHAT YOU WILL BUILD

```
NorthStar Power BI Solution:
│
├── 0. LOCAL SOURCE DATABASE
│   └── PostgreSQL 16 in Docker, database "northstar", loaded from a seeded generator
│
├── 1. DATA MODEL (Star Schema)
│   ├── dim_date         (fiscal calendar, built in M)
│   ├── dim_store        (store, district, region hierarchy)
│   ├── dim_product      (product, subcategory, category hierarchy)
│   ├── dim_customer     (customer attributes)
│   ├── dim_promotion    (promotion details)
│   ├── fact_sales       (grain: one row per order line item)
│   └── fact_targets     (grain: one row per store per month — revenue target)
│
├── 2. POWER QUERY TRANSFORMATIONS
│   ├── dim_date          (M code — fiscal year July 1, holidays)
│   ├── fact_sales        (native SQL source, date keys, days to ship)
│   └── All dimensions    (navigation + merges that fold to SQL, unused columns removed)
│
├── 3. DAX MEASURE LIBRARY (27 measures, in display folders)
│   ├── Sales:             Net Revenue, Total Cost, Gross Margin, Margin %, Order Count, AOV, Quantity, Return Rate
│   ├── Time Intelligence: YTD, QTD, PY, YoY Growth %, PY YTD, YTD Growth %, 3M Moving Average, R12M
│   ├── KPI:               Revenue Target, Variance, Variance %, Status (RAG)
│   └── Advanced:          Ship Date Revenue, Avg Days to Ship, % of Company, New vs Repeat, Pareto (Top 20%)
│
├── 4. REPORT (5 pages)
│   ├── Page 1: Executive Summary (CFO landing page)
│   ├── Page 2: Regional Performance (drill-through from Page 1)
│   ├── Page 3: Product Analytics
│   ├── Page 4: Customer Analytics
│   └── Page 5: Operational KPIs (shipping, returns)
│
└── 5. ROW-LEVEL SECURITY
    ├── Regional Access role (sees own region only; the CFO sees everything)
    └── Store Access role (sees own store only)
```

### Time Estimate: 9–11 hours

> **New to Power BI?** The SK-06 **Granular Labs** (`../Granular Labs/Lab_00` … `Lab_06`) walk through every part of this lab step by step, with explanations and checkpoints. Do them first, then come back here and build the whole solution in one go.

---

## PART 0: LOCAL ENVIRONMENT AND SOURCE DATABASE (45 min)

NorthStar's real source is an OLTP sales database. You will run your own copy on your laptop: **PostgreSQL 16 in Docker**, filled by a Python generator with a fixed random seed — so every trainee gets exactly the same rows and the same numbers as this lab.

**Prerequisites:** Windows 10/11 (64-bit), 8 GB RAM, Power BI Desktop (latest, from the Microsoft Store), Docker Desktop, Python 3.12 (3.11+ works). The generator uses only the Python standard library, so no packages and no virtual environment are needed.

### Step 0.1: Project folder and database container

```powershell
New-Item -ItemType Directory -Force C:\DataEngineering\sk06\data, C:\DataEngineering\sk06\scripts, C:\DataEngineering\sk06\sql, C:\DataEngineering\sk06\pbix | Out-Null
Set-Location C:\DataEngineering\sk06
```

Create `C:\DataEngineering\sk06\docker-compose.yml`:

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

Host port **5434** is used so this container never clashes with the SK-02 (5432) or SK-03 (5433) databases.

### Step 0.2: Generate the NorthStar data

Save as `C:\DataEngineering\sk06\scripts\generate_northstar.py`:

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

Run it from `C:\DataEngineering\sk06`:

```powershell
python scripts\generate_northstar.py --out data
```

**Expected output** (about 15 seconds; your folder path is printed on the first line):

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

### Step 0.3: Create and load the source database

Save as `C:\DataEngineering\sk06\sql\01_create_and_load_northstar.sql`:

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

Start the database and load it:

```powershell
Set-Location C:\DataEngineering\sk06
docker compose up -d
docker exec sk06-postgres pg_isready -U postgres
docker exec sk06-postgres psql -U postgres -d northstar -f /sql/01_create_and_load_northstar.sql
```

**Expected output** (end of the script — the row-count check):

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

### Step 0.4: Control totals (your answer key)

Every number in this lab can be checked against SQL run directly on the source. Run:

```powershell
docker exec -it sk06-postgres psql -U postgres -d northstar
```

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

### Step 0.5: Power BI Desktop settings and connection parameters

1. **File → Options and settings → Options → Global → Data Load** → uncheck **Auto date/time for new files** (Part 6, Optimization 5 explains why). Also uncheck it under **Current File → Data Load**.
2. Save a new file as `C:\DataEngineering\sk06\pbix\NorthStar_Sales.pbix`.
3. **Home → Transform data** → **Manage Parameters → New Parameter**, create two **Text** parameters:

| Name | Current value |
|---|---|
| `ServerName` | `localhost:5434` |
| `DatabaseName` | `northstar` |

Every query in Part 2 uses these parameters, so moving the report to another server (Part 7) changes two values, not seven queries. The first time a query connects, choose **Database** credentials `postgres` / `postgres`. If Power BI says it cannot connect using an encrypted connection, click **OK** to connect unencrypted — the local container has no TLS certificate (a production server would).

---

## PART 1: DATA MODEL DESIGN (45 min)

### Step 1.1: Star Schema Architecture

**WHY this schema:** Every DAX measure in the entire report evaluates by traversing relationships from dimension tables into fact_sales. The fewer hops the better. Denormalizing the product hierarchy (category, subcategory, brand all inside dim_product rather than in separate tables) means a product filter requires one hop, not three.

```
                    dim_date
                      │ DateKey (PK)
                      │ ← fact_sales[OrderDateKey]      [ACTIVE]
                      │ ← fact_sales[ShipDateKey]       [INACTIVE]
                      │ ← fact_targets[target_date_key] [INACTIVE]
                      ↓
dim_customer ──→ fact_sales ←── dim_product
  customer_id     OrderDateKey     product_id
                  ShipDateKey
dim_store ────→   customer_id
  store_id   │    store_id
             │    product_id
dim_promotion ─→  promotion_id
  promotion_id
             │
             └──→ fact_targets
                  (separate fact: store_id + target_date_key;
                   dim_store relationship ACTIVE, dim_date relationship INACTIVE)
```

**Key design decisions documented:**
- `fact_sales` grain: one row per order line item
- `fact_targets` grain: one row per store per month (the target is stamped on the 1st of the month)
- Two date relationships from `fact_sales` to `dim_date` (order date active, ship date inactive)
- The `fact_targets` → `dim_date` relationship is also inactive; the `[Revenue Target]` measure switches it on with `USERELATIONSHIP` (Step 3.3). This keeps date filtering of targets explicit and inside one measure.
- `dim_product` is fully denormalised — subcategory and category columns live in `dim_product`, not in separate tables
- `dim_store` is fully denormalised — district and region live in `dim_store`
- Integer keys on all tables (the source ids are already integers, and the date keys are `YYYYMMDD` integers) — not text keys, which compress worse in VertiPaq
- All relationships are one-to-many with **single** cross-filter direction (dimension → fact)

---

## PART 2: POWER QUERY (M) — DATA PREPARATION (75 min)

Create each query with **Home → New Source → Blank Query**, open **Advanced Editor**, paste the code, then rename the query (Query Settings → Name) exactly as shown.

### Step 2.1: Date Dimension (Complete M Code)

**WHY build in M (not import from SQL or use auto date/time):** NorthStar's fiscal year starts July 1, and the source database has no date table. Building it in M gives the BI team full ownership of the fiscal calendar logic without a database dependency.

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

    // ── US Federal Holidays (static list for the data range) ──
    Holidays = #table(
        type table [HolidayDate = date, HolidayName = text],
        {
            {#date(2023, 7, 4),  "Independence Day"},
            {#date(2023, 11, 23),"Thanksgiving Day"},
            {#date(2023, 12, 25),"Christmas Day"},
            {#date(2024, 1, 1),  "New Year's Day"},
            {#date(2024, 7, 4),  "Independence Day"},
            {#date(2024, 11, 28),"Thanksgiving Day"},
            {#date(2024, 12, 25),"Christmas Day"},
            {#date(2025, 1, 1),  "New Year's Day"},
            {#date(2025, 7, 4),  "Independence Day"},
            {#date(2025, 11, 27),"Thanksgiving Day"},
            {#date(2025, 12, 25),"Christmas Day"},
            {#date(2026, 1, 1),  "New Year's Day"}
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

**Check:** the query returns **2,922 rows** (1 Jan 2020 – 31 Dec 2027). Filter `Date` = 2025-07-01: `Fiscal Year` = 2026, `Fiscal Month Number` = 1, `Fiscal Quarter` = FQ1, `DateKey` = 20250701.

### Step 2.2: fact_sales — Native SQL Source

**WHY use a native SQL query as the source:** the JOIN of orders + order_lines + products and the revenue arithmetic run **inside PostgreSQL**, and only the finished, narrow result crosses the network. Power BI never imports the three raw tables (or the audit columns `created_by` and `source_system`). Note what this does to **query folding**: the native query itself runs on the server, but Power Query generally cannot fold the *later* steps of a native-query source back into SQL — so those steps run locally in the mashup engine. Keep them to cheap row-by-row work (as below), or move the logic into the SQL.

```m
// ═══════════════════════════════════════════════════════════════
// fact_sales — Sales Fact Table (grain: one row per order line)
// The joins and revenue arithmetic run inside PostgreSQL
// ═══════════════════════════════════════════════════════════════
let
    Source = PostgreSQL.Database(
        ServerName,
        DatabaseName,
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
                    CASE WHEN o.is_returned = 1 THEN 1 ELSE 0 END       AS is_returned
                FROM   sales.orders o
                JOIN   sales.order_lines ol ON o.order_id = ol.order_id
                JOIN   products.products p  ON ol.product_id = p.product_id
                WHERE  o.order_date >= DATE '2021-01-01'
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

    // Days between order and shipment (null = never shipped)
    AddDaysToShip   = Table.AddColumn(AddShipDateKey, "days_to_ship",
                          each if [ship_date] = null then null
                               else Duration.Days([ship_date] - [order_date]),
                          Int64.Type),

    // Gross margin calculated in Power Query (not a DAX calculated column —
    // this is a fixed per-row value, not context-sensitive)
    AddMargin       = Table.AddColumn(AddDaysToShip, "gross_margin",
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
                          {"promotion_id",    Int64.Type},
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

The first refresh shows a **Native Database Query** approval prompt — click **Run**. **Check:** after Close & Apply, `fact_sales` has **310,024 rows**.

### Step 2.3: Dimensions and fact_targets — Navigation + Merges (these fold)

Unlike Step 2.2, these queries start from the database's **navigation table**, so every step (select, merge, expand) can fold into one SQL statement. Right-click the last step → **View Native Query** to see the SQL that PostgreSQL actually runs.

```m
// dim_store — store → district → region, flattened
let
    Source         = PostgreSQL.Database(ServerName, DatabaseName),
    Stores         = Source{[Schema = "sales", Item = "stores"]}[Data],
    Districts      = Source{[Schema = "sales", Item = "districts"]}[Data],
    Regions        = Source{[Schema = "sales", Item = "regions"]}[Data],
    JoinDistrict   = Table.NestedJoin(Stores, {"district_id"}, Districts, {"district_id"},
                         "District", JoinKind.Inner),
    ExpandDistrict = Table.ExpandTableColumn(JoinDistrict, "District",
                         {"district_name", "region_id"}, {"district_name", "region_id"}),
    JoinRegion     = Table.NestedJoin(ExpandDistrict, {"region_id"}, Regions, {"region_id"},
                         "Region", JoinKind.Inner),
    ExpandRegion   = Table.ExpandTableColumn(JoinRegion, "Region", {"region_name"}, {"region_name"}),
    Selected       = Table.SelectColumns(ExpandRegion, {
                         "store_id", "store_name", "city", "state", "latitude", "longitude",
                         "district_name", "region_name"})
in
    Selected
```

```m
// dim_product — product → subcategory → category, flattened
let
    Source         = PostgreSQL.Database(ServerName, DatabaseName),
    Products       = Source{[Schema = "products", Item = "products"]}[Data],
    Subcategories  = Source{[Schema = "products", Item = "subcategories"]}[Data],
    Categories     = Source{[Schema = "products", Item = "categories"]}[Data],
    JoinSub        = Table.NestedJoin(Products, {"subcategory_id"}, Subcategories, {"subcategory_id"},
                         "Sub", JoinKind.Inner),
    ExpandSub      = Table.ExpandTableColumn(JoinSub, "Sub",
                         {"subcategory_name", "category_id"}, {"subcategory_name", "category_id"}),
    JoinCat        = Table.NestedJoin(ExpandSub, {"category_id"}, Categories, {"category_id"},
                         "Cat", JoinKind.Inner),
    ExpandCat      = Table.ExpandTableColumn(JoinCat, "Cat", {"category_name"}, {"category_name"}),
    Selected       = Table.SelectColumns(ExpandCat, {
                         "product_id", "product_name", "brand", "subcategory_name", "category_name",
                         "list_price"})
in
    Selected
```

```m
// dim_customer — the email column is deliberately NOT imported (personal data, not needed)
let
    Source       = PostgreSQL.Database(ServerName, DatabaseName),
    Customers    = Source{[Schema = "sales", Item = "customers"]}[Data],
    Selected     = Table.SelectColumns(Customers, {
                       "customer_id", "first_name", "last_name", "customer_segment",
                       "registration_date", "city", "state"}),
    AddFullName  = Table.AddColumn(Selected, "customer_name",
                       each [first_name] & " " & [last_name], type text)
in
    AddFullName
```

```m
// dim_promotion
let
    Source     = PostgreSQL.Database(ServerName, DatabaseName),
    Promotions = Source{[Schema = "sales", Item = "promotions"]}[Data],
    Selected   = Table.SelectColumns(Promotions, {
                     "promotion_id", "promotion_name", "promotion_type", "discount_pct",
                     "start_date", "end_date"})
in
    Selected
```

```m
// fact_targets — one row per store per month
let
    Source    = PostgreSQL.Database(ServerName, DatabaseName),
    Targets   = Source{[Schema = "finance", Item = "revenue_targets"]}[Data],
    AddKey    = Table.AddColumn(Targets, "target_date_key",
                    each Date.Year([target_month]) * 10000
                         + Date.Month([target_month]) * 100
                         + Date.Day([target_month]),
                    Int64.Type),
    Selected  = Table.SelectColumns(AddKey, {"store_id", "target_date_key", "revenue_target"}),
    Typed     = Table.TransformColumnTypes(Selected, {{"revenue_target", Currency.Type}})
in
    Typed
```

**Check after Close & Apply** (Table view, row count in the status bar):

| Table | Rows |
|---|---|
| `dim_date` | 2,922 |
| `dim_store` | 85 |
| `dim_product` | 12,000 |
| `dim_customer` | 30,000 |
| `dim_promotion` | 19 |
| `fact_sales` | 310,024 |
| `fact_targets` | 3,060 |

### Step 2.4: Relationships, date table and sort order

In **Model view**, delete any relationship Power BI auto-detected, then create these (Manage relationships → New). All are **many-to-one, single direction**:

| From (many side) | To (one side) | Active? |
|---|---|---|
| `fact_sales[OrderDateKey]` | `dim_date[DateKey]` | Yes |
| `fact_sales[ShipDateKey]` | `dim_date[DateKey]` | **No** (inactive) |
| `fact_sales[store_id]` | `dim_store[store_id]` | Yes |
| `fact_sales[product_id]` | `dim_product[product_id]` | Yes |
| `fact_sales[customer_id]` | `dim_customer[customer_id]` | Yes |
| `fact_sales[promotion_id]` | `dim_promotion[promotion_id]` | Yes |
| `fact_targets[store_id]` | `dim_store[store_id]` | Yes |
| `fact_targets[target_date_key]` | `dim_date[DateKey]` | **No** (inactive) |

Then:

1. Select `dim_date` → **Table tools → Mark as date table** → date column `Date`. Time-intelligence functions (Step 3.2) require this.
2. **Sort by column** (select the column → Column tools → Sort by column). A text column can only be sorted by a column that has exactly one value for each of its values:
   - `Year-Month` → by `Year-Month Sort`
   - `Month Name` and `Month Short` → by `Fiscal Month Number` (NorthStar reads July first)
   - `Day Name` → by `Day of Week`
3. Hide the key columns of both fact tables (`OrderDateKey`, `ShipDateKey`, `store_id`, `product_id`, `customer_id`, `promotion_id`, `target_date_key`) — report builders should pick attributes from dimensions.
4. Set `dim_store[latitude]` / `[longitude]` **Data category** to Latitude / Longitude, and `dim_store[city]` to City.

---

## PART 3: DAX MEASURE LIBRARY (90 min)

Create a measure table first: **Home → Enter data** → name it `_Measures` → Load. Create each measure below as a **separate measure** in `_Measures` (right-click → New measure; paste one `Name = expression` at a time). Put each in the **display folder** named in its block heading (Model view → select measure → Properties → Display folder). Folders give the logical grouping in the Fields pane without putting brackets in measure names (a measure name containing `[` or `]` cannot be referenced cleanly in DAX).

### Step 3.1: Base Measures (display folder `Sales`)

```dax
// Primary revenue metric — the CFO defines revenue as net of discounts
Net Revenue =
SUM(fact_sales[net_revenue])

Total Cost =
SUM(fact_sales[total_cost])

Gross Margin =
[Net Revenue] - [Total Cost]

// WHY DIVIDE not "/": DIVIDE handles division by zero gracefully.
// BLANK() is the correct alternate result — not 0 (which would
// incorrectly suggest a 0% margin rather than no data).
Gross Margin % =
DIVIDE([Gross Margin], [Net Revenue], BLANK())

Order Count =
DISTINCTCOUNT(fact_sales[order_id])

Average Order Value =
DIVIDE([Net Revenue], [Order Count], BLANK())

Quantity Sold =
SUM(fact_sales[quantity])

Return Rate =
VAR TotalOrders    = DISTINCTCOUNT(fact_sales[order_id])
VAR ReturnedOrders =
    CALCULATE(
        DISTINCTCOUNT(fact_sales[order_id]),
        fact_sales[is_returned] = 1
    )
RETURN DIVIDE(ReturnedOrders, TotalOrders, BLANK())
```

**Check** (slicer `dim_date[Fiscal Year]` = 2026): Net Revenue **$11,862,350.26**, Total Cost $7,655,104.68, Gross Margin % **35.47%**, Order Count **63,879**, Average Order Value **$185.70**, Quantity Sold 179,431, Return Rate **3.93%**. These match the SQL control totals in Step 0.4.

### Step 3.2: Time Intelligence Measures (display folder `Time Intelligence`)

```dax
// ── WHY the "6/30" fiscal year end parameter ───────────────────
// NorthStar's fiscal year ends June 30. Without this parameter,
// DATESYTD starts accumulating from January 1. With it, the YTD
// resets at the correct fiscal year boundary.

Revenue YTD =
CALCULATE(
    [Net Revenue],
    DATESYTD(dim_date[Date], "6/30")
)

// Fiscal quarters (Jul–Sep, Oct–Dec, ...) line up with calendar
// quarters, so DATESQTD needs no fiscal parameter.
Revenue QTD =
CALCULATE(
    [Net Revenue],
    DATESQTD(dim_date[Date])
)

// Prior year same period — the CFO's most-requested benchmark
Revenue PY =
CALCULATE(
    [Net Revenue],
    SAMEPERIODLASTYEAR(dim_date[Date])
)

// WHY BLANK() not 0: If there is no prior year data (first year),
// showing 0% growth is misleading. BLANK() correctly shows no comparison.
YoY Growth % =
VAR CurrentRevenue   = [Net Revenue]
VAR PriorYearRevenue = [Revenue PY]
RETURN
DIVIDE(
    CurrentRevenue - PriorYearRevenue,
    PriorYearRevenue,
    BLANK()
)

// YTD of the prior year (for YTD growth tracking): shift the dates back
// a year, then let [Revenue YTD] apply the fiscal YTD logic
Revenue PY YTD =
CALCULATE(
    [Revenue YTD],
    SAMEPERIODLASTYEAR(dim_date[Date])
)

YTD Growth % =
DIVIDE(
    [Revenue YTD] - [Revenue PY YTD],
    [Revenue PY YTD],
    BLANK()
)

// 3-month moving average smooths holiday/promotion spikes.
// Total revenue of the last 3 months divided by 3 = average MONTHLY revenue.
// (Averaging over the dates of the window would give average DAILY revenue.)
Revenue 3M Moving Avg =
VAR Last3Months =
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -3, MONTH)
RETURN
DIVIDE(CALCULATE([Net Revenue], Last3Months), 3)

// Rolling 12 months (does not reset at year end — unlike YTD)
Revenue R12M =
CALCULATE(
    [Net Revenue],
    DATESINPERIOD(dim_date[Date], MAX(dim_date[Date]), -12, MONTH)
)
```

**Check** (matrix: rows `dim_date[Year-Month]`, values the measures above):

| Year-Month | Net Revenue | Revenue YTD | Revenue PY | YoY Growth % | Revenue 3M Moving Avg |
|---|---|---|---|---|---|
| 2025-07 | $1,032,585.71 | $1,032,585.71 | $881,559.21 | 17.13% | $938,066.03 |
| 2025-12 | $1,349,327.08 | $6,475,316.27 | $1,291,731.85 | 4.46% | $1,188,982.33 |

- In July (first fiscal month) `Revenue YTD` equals `Net Revenue` — the fiscal reset works.
- At 2025-12: `Revenue PY YTD` = $5,866,831.07 and `YTD Growth %` = **10.37%**; `Revenue QTD` = $3,566,946.98.
- Fiscal Year 2026 total: `YoY Growth %` = **10.01%**; at 2026-06 `Revenue R12M` = $11,862,350.26 (the whole of FY2026).
- The 3-month average for the first two months of data (2023-07, 2023-08) is understated because fewer than 3 months exist — read the line from FY2025 onwards.

### Step 3.3: KPI Measures (display folder `KPI`)

```dax
// Revenue Target — from fact_targets
// WHY USERELATIONSHIP: fact_targets[target_date_key] is linked to
// dim_date via an INACTIVE relationship (Step 2.4). USERELATIONSHIP
// activates it within this measure only, so the year/month slicers that
// filter fact_sales also filter fact_targets.
Revenue Target =
CALCULATE(
    SUM(fact_targets[revenue_target]),
    USERELATIONSHIP(fact_targets[target_date_key], dim_date[DateKey])
)

Revenue Variance =
[Net Revenue] - [Revenue Target]

Revenue Variance % =
DIVIDE([Revenue Variance], [Revenue Target], BLANK())

// Traffic light for conditional formatting on KPI cards
// Returns 1 (green), 0 (yellow), -1 (red) — or BLANK when there is no target
Revenue Status =
VAR VarPct = [Revenue Variance %]
RETURN
IF(
    ISBLANK(VarPct),
    BLANK(),                // no target in this context: no colour, not "green"
    SWITCH(
        TRUE(),
        VarPct >= 0,     1,     // Green: at or above target
        VarPct >= -0.10, 0,     // Yellow: within 10% below target
        -1                      // Red: more than 10% below target
    )
)
```

**Check** (Fiscal Year 2026, matrix rows `dim_store[region_name]`):

| Region | Net Revenue | Revenue Target | Variance % | Status |
|---|---|---|---|---|
| Northeast | $3,346,490.01 | $3,239,600.00 | 3.30% | 1 |
| Southeast | $3,103,316.68 | $3,191,600.00 | -2.77% | 0 |
| Midwest | $3,053,466.54 | $3,263,200.00 | -6.43% | 0 |
| West | $2,359,077.03 | $2,688,900.00 | -12.27% | -1 |
| **Total** | **$11,862,350.26** | **$12,383,300.00** | **-4.21%** | **0** |

### Step 3.4: Advanced Measures (display folders `Shipping`, `Analysis`, `Advanced`)

```dax
// ── Ship Date Revenue (inactive relationship activation) ────────
// The logistics team reports by ship date, not order date.
// USERELATIONSHIP activates the inactive ShipDateKey → dim_date
// relationship for this measure only.
Revenue by Ship Date =
CALCULATE(
    [Net Revenue],
    USERELATIONSHIP(fact_sales[ShipDateKey], dim_date[DateKey])
)

// Average days from order to shipment, per order line.
// days_to_ship was computed in Power Query (Step 2.2); unshipped
// lines have a null value, and AVERAGE ignores blanks.
Avg Days to Ship =
AVERAGE(fact_sales[days_to_ship])

// ── Revenue % of Company Total ──────────────────────────────────
// REMOVEFILTERS(dim_store) removes every store/district/region filter,
// so the denominator is the whole company. Filters on other tables —
// the Fiscal Year slicer on dim_date, a category filter — are PRESERVED.
// (Removing only region_name and store_name would break as soon as
// someone filters on district_name or city.)
Revenue % of Company Total =
DIVIDE(
    [Net Revenue],
    CALCULATE(
        [Net Revenue],
        REMOVEFILTERS(dim_store)
    ),
    BLANK()
)

// ── New vs Repeat Customer Revenue ─────────────────────────────
// "New" = the customer registered inside the selected period
New Customer Revenue =
CALCULATE(
    [Net Revenue],
    FILTER(
        dim_customer,
        dim_customer[registration_date] >= MIN(dim_date[Date]) &&
        dim_customer[registration_date] <= MAX(dim_date[Date])
    )
)

Repeat Customer Revenue =
[Net Revenue] - [New Customer Revenue]

Repeat Customer % =
DIVIDE([Repeat Customer Revenue], [Net Revenue], BLANK())

// ── Pareto: share of revenue from the top 20% of products ──────
Top 20% Products Revenue % =
VAR ProductsInScope = VALUES(dim_product[product_id])
VAR TopCount        = ROUNDUP(COUNTROWS(ProductsInScope) * 0.2, 0)
VAR TopProducts     = TOPN(TopCount, ProductsInScope, [Net Revenue], DESC)
RETURN
DIVIDE(CALCULATE([Net Revenue], TopProducts), [Net Revenue], BLANK())
```

**Check** (Fiscal Year 2026 unless stated): Revenue by Ship Date **$11,709,129.07** (orders placed in late June 2025 and shipped in July count in FY2026; orders never shipped or shipped after June 2026 do not). Avg Days to Ship **3.42** (all years and FY2026 alike). Revenue % of Company Total for Northeast **28.21%**. New Customer Revenue $2,236,771.55, Repeat Customer % **81.14%**. Top 20% Products Revenue % **80.07%** — NorthStar's catalogue follows the 80/20 rule. `Customer Segment` needs no measure: put `dim_customer[customer_segment]` on a slicer or axis and every measure splits by it.

---

## PART 4: ROW-LEVEL SECURITY (30 min)

### Step 4.1: Security Mapping Tables

Create these two queries (Blank Query → Advanced Editor), load them, and **do not relate them** to any other table — the role filters look them up with `LOOKUPVALUE`. Hide both tables in Report view.

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

### Step 4.2: RLS Role Configuration

**In Power BI Desktop → Modeling → Manage Roles** (in the newer editor, switch to the **DAX editor** to paste an expression):

**Role: "Regional Access"** — table filter on `dim_store`:
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

**Role: "Store Access"** — table filter on `dim_store`:
```dax
// Even more restrictive — store managers see only their own store
dim_store[store_id] =
    LOOKUPVALUE(
        security_user_store[store_id],
        security_user_store[user_email],
        USERPRINCIPALNAME()
    )
```

### Step 4.3: Testing RLS

**In Power BI Desktop → Modeling → View as**, with the `Fiscal Year` slicer on 2026:

1. Tick **Other user** = `ne.manager@northstar.com` and role **Regional Access**. Net Revenue shows **$3,346,490.01**, and a region visual shows only Northeast.
2. Check that *every* measure follows: Revenue Target $3,239,600.00, Order Count only Northeast's orders. Note `Revenue % of Company Total` now shows **100%** — RLS filters cannot be removed by `REMOVEFILTERS`, so "the company" is whatever the user may see. Decide with the CFO whether that is the wanted behaviour.
3. Other user = `cfo@northstar.com`, role Regional Access → the full **$11,862,350.26**.
4. Other user = `boston101.manager@northstar.com`, role **Store Access** → **$80,855.91** (store 101, NorthStar Boston #101 only).
5. **Critical negative test:** Other user = `intruder@northstar.com` with Regional Access → every visual is **blank**. If anything shows, the rule has a hole.
6. In the Service, also verify that a Northeast user cannot see Southeast data by changing URL filter parameters — RLS is enforced by the engine, so URL filters can only narrow what the role allows.

---

## PART 5: REPORT DESIGN (60 min)

### Step 5.1: Page 1 — Executive Summary Layout

The card subtitles and the drill-through header need three small helper measures (display folders `Time Intelligence` and `Analysis`):

```dax
Gross Margin % PY =
CALCULATE([Gross Margin %], SAMEPERIODLASTYEAR(dim_date[Date]))

Order Count YoY % =
VAR CurrentOrders = [Order Count]
VAR PriorOrders   = CALCULATE([Order Count], SAMEPERIODLASTYEAR(dim_date[Date]))
RETURN DIVIDE(CurrentOrders - PriorOrders, PriorOrders, BLANK())

Selected Region =
SELECTEDVALUE(dim_store[region_name], "All regions")
```

```
LAYOUT SPECIFICATION (numbers shown for Fiscal Year = 2026):
─────────────────────────────────────────────────────────────────

ROW 1 — FILTER BAR (top of page, full width)
  [Fiscal Year slicer — dropdown]  [Fiscal Quarter slicer — buttons]  [Region slicer — buttons]

ROW 2 — KPI CARDS (4 equally-spaced cards)
  Card 1: Net Revenue ($11.86M)
    Subtitle: vs Prior Year ($10.78M)
    Color: Green if YoY > 0, Red if YoY < 0
    Trend axis: Monthly revenue (line sparkline)

  Card 2: YoY Growth (+10.0%)
    Conditional icon: ▲ green / ▼ red
    Reference label: "Target: 5.0%"

  Card 3: Gross Margin % (35.5%)
    Conditional color: Green if > 35%, Yellow if 30-35%, Red if < 30%
    Subtitle: vs PY (35.5%)

  Card 4: Order Count (63.9K)
    Subtitle: +8.7% vs PY (PY = 58,745)
    Trend axis: Monthly order count

ROW 3 — LEFT CHART: Revenue Trend (60% width)
  Line chart: X axis = Year-Month (sorted by Year-Month Sort)
    (Do not use Month Name here: across years it repeats, so it cannot be
     sorted by Year-Month Sort — Power BI refuses that sort.)
  Lines:
    - Net Revenue (solid blue #2196F3)
    - Revenue PY (dashed grey #9E9E9E)
    - Revenue 3M Moving Avg (dotted orange #FF9800)
    - Revenue Target (dashed red)
  Title: "Revenue Trend — Current Year vs Prior Year"

ROW 3 — RIGHT CHART: Revenue by Region (40% width)
  Clustered bar chart (horizontal — regions are long text)
  Bars: Net Revenue (blue) + Revenue Target (outlined)
  Conditional bar color (fx, rules on Revenue Status): Green if 1, Amber if 0, Red if -1
  Drill-through: Right-click any bar → Drill through → Page 2

ROW 4 — LEFT TABLE: Top 10 Products (60% width)
  Table columns: product_name, Net Revenue, Revenue % of Company Total,
                 YoY Growth %, Gross Margin %
  Visual filter: product_name → Top N = 10 by Net Revenue
  Conditional formatting:
    - YoY Growth %: icon set (▲ green, ► yellow, ▼ red)
    - Gross Margin %: data bars (light blue gradient)
  Sort: by Net Revenue descending

ROW 4 — RIGHT CHART: Revenue by Category (40% width)
  Donut chart: 5 segments (one per category_name)
  Legend: visible, positioned right
  Labels: show % of total (Sports & Outdoors is the largest at 27.3%)
```

### Step 5.2: Drill-Through Configuration for Page 2

On **Page 2 (Regional Performance)**:
1. Add `dim_store[region_name]` to the **Drill-through** field well
2. This enables right-click → "Drill through" from any visual on Page 1 that has region context
3. Power BI adds a Back button automatically when you set up drill-through (otherwise Insert → Buttons → Back)
4. Add a card with `[Selected Region]` as the page header. Drilling from the Northeast bar (FY2026) shows **Northeast** and 24 stores, led by NorthStar Newark #123 ($203,125.04).
5. Page 2 should show store-level breakdowns (table of `store_name` with Net Revenue, Revenue Target, Revenue Variance %), a map visual (latitude/longitude from `dim_store`, bubble size = Net Revenue), and store-vs-target comparison. If the map is blank, enable **File → Options → Security → Use Map and Filled Map visuals**.

Pages 3–5 use the same measures: Product Analytics (category → subcategory → product matrix, Top 20% Products Revenue %), Customer Analytics (customer_segment, New vs Repeat Customer Revenue), Operational KPIs (Revenue by Ship Date, Avg Days to Ship, Return Rate).

---

## PART 6: PERFORMANCE OPTIMIZATION (30 min)

### Step 6.1: Performance Analyzer Walkthrough

```
HOW TO USE PERFORMANCE ANALYZER:
1. Power BI Desktop → Optimize (older versions: View) → Performance Analyzer
2. Click "Start recording"
3. Interact with the Executive Summary page (change slicer, click a bar)
4. Click "Stop"
5. Expand each visual to see three time components:
   - DAX query: time for the engine to calculate measures
   - Direct query: time waiting for the source database (0 in Import mode)
   - Other: visual rendering and waiting for other visuals

TARGET:
  < 200ms  = Excellent (don't touch)
  200-500ms = Good (monitor)
  500ms-2s  = Investigate the DAX
  > 2s      = Must optimize before production

WHAT TO DO WITH A SLOW VISUAL:
1. Click "Copy query" next to the slow visual
2. Paste into DAX query view (or DAX Studio, free from daxstudio.org)
3. In DAX Studio, run with Server Timings enabled
4. Look for: high FE (formula engine) time = slow DAX
           many SE (storage engine) queries = DAX iterating row by row
```

### Step 6.2: Model Optimization Checklist

```
OPTIMIZATION 1: Remove Unused Columns ─────────────────────────
Impact: Largest single optimization — commonly reduces model size 30-70%
Action: In Power Query, remove all columns not referenced in:
  • Any measure
  • Any calculated column
  • Any visual field well
  • Any relationship

Columns this lab never imports:
  - created_by, source_system (audit/pipeline fields on sales.orders)
  - email (personal data on sales.customers — also a privacy win)
  - order_line_id, line_number, open_date, unit_cost (not used by any visual)
Typical candidates in real sources: modified_by, etl_timestamp, batch_number,
  internal_notes, session_id, ip_address

OPTIMIZATION 2: Integer Keys ───────────────────────────────────
Impact: Text keys compress far worse than integers
BEFORE: customer_id = "CUST-NE-00042837" (text, ~20 bytes, unique per row)
AFTER:  customer_id = 42837 (whole number)
Action: Keep or create integer keys (NorthStar's source already uses them;
        dates use YYYYMMDD integer keys)

OPTIMIZATION 3: Measures over Calculated Columns ───────────────
Impact: Reduces model size; measures compute only when needed
Rule: If the value changes based on slicer selection → MUST be measure
If the value is a fixed row attribute → consider calculated column
      (but Power Query is usually better)
BEFORE (calculated column — stored per row):
  margin_pct = DIVIDE(fact_sales[net_revenue] - fact_sales[total_cost],
                      fact_sales[net_revenue])
AFTER (measure — no storage cost, and correct at every level):
  Gross Margin % = DIVIDE([Gross Margin], [Net Revenue], BLANK())

OPTIMIZATION 4: Variables in DAX ──────────────────────────────
Impact: Avoids evaluating the same expression twice
Rule: If a sub-expression is referenced more than once → use VAR
BEFORE (evaluates [Revenue PY] twice):
  YoY % = DIVIDE([Net Revenue] - [Revenue PY], [Revenue PY])
AFTER (evaluates [Revenue PY] once):
  YoY % = VAR CurrentRev = [Net Revenue]
          VAR PriorRev   = [Revenue PY]
          RETURN DIVIDE(CurrentRev - PriorRev, PriorRev, BLANK())

OPTIMIZATION 5: Disable Auto Date/Time ─────────────────────────
Impact: Removes hidden date tables (one per date column in the model)
Action: File → Options → Data Load → Uncheck "Auto date/time"
        (Then mark your custom dim_date as the date table)
```

---

## PART 7: DEPLOYMENT (20 min)

### Step 7.1: Publish to Power BI Service

Publishing needs a work or school account with Power BI access. **Deployment pipelines** additionally need Premium Per User, Premium or Fabric capacity (a Fabric trial works). Because this lab's source is a PostgreSQL container on your laptop, **scheduled refresh in the Service needs an on-premises data gateway** installed on a machine that can reach it; without a gateway you can still publish and view the imported data.

```
DEPLOYMENT PIPELINE SETUP:

[Dev Workspace]    [Test Workspace]   [Production Workspace]
  ↑                    ↑                    ↑
Developers publish   QA validates data    End users consume
No restrictions      and performance      Viewer access only
                                          2 people can deploy

STEPS:
1. Create 3 workspaces: NorthStar-Dev, NorthStar-Test, NorthStar-Prod
2. Power BI Service → Deployment pipelines → Create a pipeline
3. Assign workspaces: Dev → Test → Prod
4. Add deployment rules (parameter rules on ServerName / DatabaseName):
   - Dev:  ServerName = northstar-dev-db:5432
   - Test: ServerName = northstar-test-db:5432
   - Prod: ServerName = northstar-prod-db:5432

PUBLISH WORKFLOW:
Developer → Publish to Dev workspace → Deploy to Test →
QA validates → Team lead approves → Deploy to Production

SECURITY (Production semantic model → Security):
- Add the regional managers' group to "Regional Access"
- Add the store managers' group to "Store Access"
- Workspace Admins/Members/Contributors are NOT filtered by RLS —
  give report consumers Viewer access or an app

REFRESH SCHEDULE (Production):
- Daily at 5:00 AM (before business hours)
- Failure notification: data-team@northstar.com
- Timeout: 2 hours
- Gateway: NorthStar-PBI-Gateway (on-premises PostgreSQL connection)
```

---

## REFLECTION QUESTIONS

Answer these after completing the lab:

1. The `fact_targets` table links to `dim_date` via an inactive relationship. The CFO asks you to create a Revenue vs Target visual that responds to the Year and Quarter slicers. You use USERELATIONSHIP in the Revenue Target measure. But when you add both Revenue and Revenue Target to the same visual, Revenue responds to the slicer but Revenue Target always shows the full total. What is most likely wrong?

2. The date dimension M code uses `Date.DayOfWeek([Date], Day.Monday)` to compute the day of week. What is the difference between `Day.Monday` and `Day.Sunday` as the second parameter, and which one is correct for a US retailer reporting dashboard?

3. A regional manager complains: "When I open the report, I see all regions' data, not just mine." You know RLS is configured correctly because "View as" in Desktop shows the correct filtering. What are the 3 most likely causes of this issue in the deployed Power BI Service environment?

4. The Performance Analyzer shows that the "Revenue Trend" line chart takes 4.2 seconds: DAX query = 3.8s, Other = 0.4s. You open the DAX in DAX Studio with Server Timings. You see 12 storage engine (SE) queries and 1 formula engine (FE) pass. What does the high SE query count suggest about the DAX measure, and what pattern would you change to reduce it?

5. The CFO asks you to add a "Budget Variance by Product Category" visual, but the `fact_targets` table only has targets per store and month — not per product category. Describe two approaches to handle this (one using the existing data model, one requiring a data model change), and explain the trade-off.
