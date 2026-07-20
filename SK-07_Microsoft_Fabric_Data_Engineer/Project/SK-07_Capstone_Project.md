# SK-07 CAPSTONE PROJECT: E-Commerce Analytics Platform for TrendMart

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **TrendMart**, an e-commerce company processing **500,000 orders per day** across 12 product categories and 3 fulfilment regions. TrendMart uses four systems: Shopify (orders and products), Stripe (payments and refunds), Google Analytics 4 (web traffic via BigQuery export), and Zendesk (customer support tickets).

Currently, each system is siloed: the CMO asks the data team for customer lifetime value analysis, which requires joining Shopify orders, Stripe refunds, and Zendesk tickets manually in Excel. This takes a data analyst 3 days per report. The CEO wants a self-service analytics platform where any analyst can answer cross-system questions without writing SQL.

**Additional requirements from the CTO:**
- A real-time orders dashboard showing orders per minute and revenue run-rate, updated within 30 seconds of each order
- The marketing team must be able to blend their campaign data (managed in Excel on SharePoint) with the Gold Lakehouse data without engineering help
- Bronze and Silver data must not be accessible to business analysts — only Gold data exposed via the Warehouse

---

## SOURCE DATA DESCRIPTION

Generate realistic synthetic data for all source tables.

| Source System | Tables | Volume |
|---|---|---|
| Shopify | `orders`, `order_items`, `products`, `customers` | 500K orders/day |
| Stripe | `payments`, `refunds`, `disputes` | ~510K events/day |
| GA4 (BigQuery export) | `events`, `sessions` | 2M events/day |
| Zendesk | `tickets`, `ticket_comments`, `ticket_tags` | 5K tickets/day |

**Synthetic data requirements:**
- 3 years of historical data (daily grain)
- Embed realistic patterns: Black Friday volume spike (5× normal), weekend slowdown (−30%), seasonal category trends
- Customer repeat purchase rate of ~35% (typical e-commerce benchmark)
- Refund rate of ~8% (industry average for fashion e-commerce)
- At least 15% of tickets tagged "return_request" and linked to orders
- At least 200 customers with 10+ orders (VIP segment)
- At least 500 customers who placed exactly one order 90+ days ago and never returned (lapsed segment)

---

## PROJECT REQUIREMENTS

### Deliverable 1: Medallion Architecture Design Document

Before building, produce a written design document covering:

**Bronze layer:**
- Table schemas for all 7 source tables (column names, types, partition column, metadata columns)
- Partition strategy justification (why partition Shopify orders by order_date vs by customer_id vs by product_id?)
- Ingestion frequency and retention policy for each table

**Silver layer:**
- Deduplication strategy for each table (Shopify order_id is the natural key, but what about GA4 events which have no stable unique ID?)
- Conforming decisions: GA4 currency is always USD — Shopify supports multi-currency. How do you normalise to USD in Silver?
- Data quality rules (minimum 3 per table) with the quarantine vs drop decision for each rule

**Gold layer:**
- Complete star schema diagram showing: fact_orders, fact_support_tickets, dim_customer, dim_product, dim_date, and any bridge tables needed
- Grain for each fact table (one row per what?)
- Identification and justification of one semi-additive measure in the data model
- Z-order columns for each fact table (justify based on expected query patterns)
- The cross-domain analysis requirement: "Do customers who submit support tickets within 7 days of an order have higher or lower LTV?" — which tables need to be joined to answer this, and at what grain?

---

### Deliverable 2: Fabric Workspace and Lakehouse Provisioning

Provision the following (document each with screenshots or configuration notes):

- Two workspaces: `trendmart-data-engineering` and `trendmart-analytics`
- Three Lakehouses in `trendmart-data-engineering`: `trendmart_bronze`, `trendmart_silver`, `trendmart_gold`
- One Warehouse in `trendmart-analytics`: `trendmart_warehouse`
- OneLake shortcuts from `trendmart_gold` tables into `trendmart_warehouse`
- Workspace role assignments: document which AAD groups get which roles in each workspace and why

---

### Deliverable 3: Fabric Data Pipelines

Build a pipeline with the following characteristics:

**Main pipeline: `TrendMart_Daily_Ingestion`**
- Parameter: `processDate` (defaults to yesterday)
- Parallel Copy Activities: Shopify Orders (REST API), Stripe Payments (REST API), Zendesk Tickets (REST API)
- Sequential dependency: GA4 BigQuery export Copy runs after Shopify Copy succeeds (BigQuery export is triggered by a Shopify webhook — you must wait for it)
- After all 4 copies succeed: run `01_bronze_to_silver` notebook
- After Silver notebook succeeds: run `02_silver_to_gold` notebook
- On any failure: send Teams webhook alert with pipeline name, process_date, failing activity name, and error message
- On completion (success or failure): write a row to `trendmart_bronze.pipeline_audit_log` Delta table

**Configuration requirements:**
- Retry count 3, retry interval 90 seconds on all Copy Activities
- The Shopify API is paginated — configure pagination (next page URL: `$.pagination.next`)
- Stripe API requires an Authorization header — show how to store the API key securely (Fabric Key Vault reference or environment variable) without hardcoding in the pipeline JSON

---

### Deliverable 4: PySpark Transformation Notebooks

**Notebook 1: `01_bronze_to_silver.py`**

Required transformations for all 4 source domains:

*Shopify Orders:*
- Deduplication on order_id
- Currency normalisation: convert all non-USD amounts to USD using a static exchange rate table (hardcoded rates are fine for the capstone — document the limitation)
- Derived column: `order_value_tier` (Low: < $50, Medium: $50–$200, High: > $200)
- Quality rule: discard orders with `total_price_usd <= 0` — log to `_rejected` table
- Delta MERGE on order_id (orders can be updated — status changes from PENDING to FULFILLED)

*Stripe Payments:*
- Deduplication on payment_id
- Link to Shopify orders via `metadata.order_id` (Stripe stores the Shopify order_id in metadata — parse this field)
- Quality rule: flag payments where `amount != order amount` (tolerance ±$1 for rounding)
- replaceWhere on payment_date partition (payments for a given date are fully reloaded)

*GA4 Sessions:*
- GA4 exports one event per row — group into sessions using session_id (one row per session in Silver)
- Derive session metrics: `session_duration_minutes`, `page_view_count`, `add_to_cart_count`, `checkout_reached` (boolean), `purchase_completed` (boolean)
- These aggregations cannot be done with a simple dedup — use groupBy + agg

*Zendesk Tickets:*
- Deduplication on ticket_id
- Parse the ticket tags array: create boolean columns `has_return_tag`, `has_shipping_tag`, `has_billing_tag`
- Link to Shopify: extract order_id from ticket subject line (format: "Order #12345 issue") using regex
- Delta MERGE on ticket_id (tickets are updated as agents respond)

**Notebook 2: `02_silver_to_gold.py`**

Build the complete star schema:
- dim_customer: one row per customer (latest attributes from Silver), with derived columns: `is_repeat_customer`, `lifetime_order_count`, `first_order_date`
- dim_product: one row per product (latest attributes), with `category`, `subcategory`, `brand`
- fact_orders: grain = one row per order_item (not per order — enables product-level analysis); measures: `item_revenue_usd`, `item_cost_usd`, `discount_amount_usd`; surrogate keys joining all dimensions
- fact_support_tickets: grain = one row per ticket; measures: `resolution_hours`, `reply_count`, `has_return_tag`; links to fact_orders via order_id (many-to-many bridge is NOT required — one ticket = one primary order)

Post-transformation assertions (minimum 5):
1. fact_orders row count within expected range
2. No null customer_key in fact_orders
3. No negative item_revenue_usd
4. Total fact_orders revenue within 5% of Silver `silver_orders` total revenue (end-to-end reconciliation)
5. fact_support_tickets: no tickets with resolution_hours > 8760 (one year — data error)

OPTIMIZE and ZORDER after all Gold tables are written.

---

### Deliverable 5: Fabric Warehouse Analytical Views

Create the following views in `trendmart_warehouse` (using shortcuts to Gold Lakehouse tables):

**View 1: vw_customer_lifetime_value**
Required logic: per customer, compute total revenue, total orders, avg order value, first order date, most recent order date, days since last order, and a `customer_segment` classification (Champion: recent + frequent + high spend; At Risk: high LTV but no order in 60+ days; Lapsed: no order in 90+ days; New: first order in last 30 days).

The segment logic must be written as a CTE-based T-SQL view — not in Python or DAX.

**View 2: vw_product_performance**
Per product per month: revenue, units sold, return rate (from Stripe refunds joined to fact_orders), YoY revenue change using LAG(), and a `performance_tier` (Top 10%, Mid 10–50%, Tail 50%+) using NTILE(10) window function.

**View 3: vw_support_correlation**
This is the hardest view. Per customer, join fact_orders to fact_support_tickets to compute:
- Total orders
- Orders with a support ticket within 7 days of order date
- Revenue from orders with vs without support tickets
- Whether support-ticket customers have higher or lower LTV

This view tests your ability to write a time-bounded join in T-SQL (order_date BETWEEN ticket_date - 7 AND ticket_date + 7).

---

### Deliverable 6: Dataflows Gen2 — Marketing Campaign Data

The marketing team maintains a campaign performance Excel file on SharePoint:

| Column | Description |
|---|---|
| campaign_date | Date of the campaign (daily grain) |
| channel | Email, Paid Search, Social, Display, Affiliate |
| campaign_name | Campaign identifier |
| spend_usd | Ad spend for that day |
| impressions | Number of impressions |
| clicks | Number of clicks |
| attributed_orders | Orders attributed to this campaign (in Shopify) |
| attributed_revenue | Revenue from attributed orders |

Build a Dataflow Gen2 that:
1. Connects to the SharePoint Excel file (no gateway needed for cloud SharePoint)
2. Applies type enforcement and null handling (rows with null spend_usd → drop)
3. Computes `roas` (Return on Ad Spend = attributed_revenue / spend_usd)
4. Writes to `trendmart_gold.fact_campaign_performance` with **Merge** update method (key: campaign_date + channel + campaign_name)
5. Schedules weekly refresh (Monday 06:00 UTC)

Document how a non-technical marketing analyst would update the Excel file and trigger a manual refresh without needing engineering help.

---

### Deliverable 7: Real-Time Intelligence — Live Order Dashboard

**Eventstream setup:**
TrendMart's Shopify account publishes order creation webhooks to an Azure Event Hub ("trendmart-orders-live"). Each event contains: order_id, customer_id, total_price_usd, product_category, fulfilment_region, created_at.

1. Create an Eventstream "trendmart_live_orders" sourcing from this Event Hub
2. Add a Filter transformation: exclude orders where total_price_usd <= 0 (test orders)
3. Route to KQL Database "trendmart_realtime", table "live_orders"
4. Route ALL events (unfiltered) to Bronze Lakehouse as "live_orders_archive"

**KQL queries for the real-time dashboard (write all 4):**

Query 1 — Orders per minute (last 2 hours):
Produce a timechart showing order count per 1-minute bucket for the last 2 hours.

Query 2 — Revenue run-rate:
Compute today's total revenue so far, projected revenue for the full day (extrapolate from current hour's rate), and compare to yesterday's total. Return as a single-row table suitable for a KPI tile.

Query 3 — Orders by category (last 30 minutes):
Real-time product category breakdown — which categories are selling right now? Return as a table with category, order_count, and total_revenue.

Query 4 — Alert: order volume drop:
Return a result (non-empty table) when the orders-per-minute in the last 10 minutes is more than 40% below the rolling 60-minute average. This is the "site is down" detector.

**Build a 4-tile Fabric real-time dashboard** using these 4 queries.

---

### Deliverable 8: Governance and Access Control

Implement the following access control model and document each decision:

| Role | Workspace Access | Permitted Items |
|---|---|---|
| Data Engineers | Contributor on `trendmart-data-engineering` | All Lakehouses, all Pipelines, all Notebooks |
| Data Scientists | Viewer on `trendmart-data-engineering` | Silver Lakehouse only (via item-level share) |
| Business Analysts | Viewer on `trendmart-analytics` | Warehouse views and semantic models only |
| Marketing Team | No workspace access | Item-level share to `fact_campaign_performance` shortcut only |
| Executives | No workspace access | Item-level share to Power BI reports only |

**Documentation required:**
- Which items have sensitivity labels applied and at what level?
- Run Impact Analysis on `fact_orders` in the Gold Lakehouse. List all downstream artefacts shown.
- Write a 1-page "New Analyst Onboarding" procedure: how does a new business analyst get access to the Warehouse, and what is the escalation path if they need access to Silver data?

