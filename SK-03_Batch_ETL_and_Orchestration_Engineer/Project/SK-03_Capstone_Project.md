# SK-03 CAPSTONE PROJECT: Multi-Source Financial Data Pipeline for FinCore

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **FinCore**, a mid-size financial services firm that manages trading accounts for institutional clients. Their risk management team starts their day at **6:30 AM** reviewing overnight trading activity — but the data they analyze is assembled manually by a junior analyst who arrives at 5 AM to run a sequence of Python scripts.

This process is fragile:

- The pipeline has no retry logic. A transient Bloomberg API timeout at 5:15 AM means the risk team has no data at 6:30 AM.
- Three data sources arrive at different times. The analyst manually checks each one before proceeding — there is no automated dependency management.
- P&L calculations require joining trade data with market prices. If the market data feed is delayed, the analyst must decide whether to wait or proceed with stale prices. There is no policy.
- The pipeline has never been tested for correctness. Last quarter, a decimal precision error in the FX conversion function overstated GBP-denominated P&L by $1.2M for three days before anyone noticed.

The Head of Risk has mandated: deliver an automated, monitored, quality-gated pipeline that runs reliably before 6:30 AM every trading day.

## 

---

## DATA SOURCES

Three source feeds arrive in S3 daily after market close (5:00–6:00 PM EST). Your pipeline processes them overnight and must be complete before 6:30 AM the following day.

| Source | Format | S3 Path | Typical Arrival | Volume |
|---|---|---|---|---|
| **FIX Protocol Trade Log** | Delimited text (FIX 4.4 tags) | `s3://fincore-raw/trades/dt={date}/` | 5:15 PM | ~500K records |
| **Market Data Snapshot** | CSV (one file per asset class) | `s3://fincore-raw/market_data/dt={date}/` | 5:45 PM | ~50K records |
| **Internal Portfolio State** | JSON (exported from portfolio system) | `s3://fincore-raw/portfolio/dt={date}/` | 6:00 PM | ~8K records |

---

## PROJECT REQUIREMENTS

### Deliverable 1: PySpark ETL Job

Build a single PySpark job (`etl_job.py`) that processes all 3 sources.

**Minimum requirements:**

**FIX parsing:**
- Parse the FIX pipe-delimited format into a structured DataFrame (each tag becomes a column)
- Handle malformed lines using PERMISSIVE mode + dead-letter routing
- Map FIX Side codes (1/2) to human-readable BUY/SELL
- Parse TransactTime from FIX timestamp format (YYYYMMDD-HH:MM:SS)

**Market data processing:**
- Read all CSV files for the date, union them into a single DataFrame
- Handle NULL close_price (route to dead-letter, do not use stale prices — this is a financial requirement)
- Validate close_price > 0 for all non-null values

**Portfolio enrichment:**
- Read JSON with explicit schema, extract the nested `metadata.risk_limits` fields
- Join portfolio positions with market data on symbol (after symbol normalization)
- Calculate mark-to-market value: `position_quantity × close_price` in USD (apply FX conversion for non-USD positions)

**P&L calculation (join trades + market data):**
- For each trade, look up the market close_price on the same date
- Calculate realized_pnl: `(close_price - average_cost) × quantity × side_multiplier`
- Handle the case where no market price is available for a symbol (flag as `pnl_unresolvable=True`, route to a separate output)

**Output:** Write 3 separate Parquet outputs:
- `processed/trades/dt={date}/` — clean trade records
- `processed/portfolio_pnl/dt={date}/` — portfolio-level P&L
- `rejected/dt={date}/` — all rejected records with reason

**Quality metrics:** Write a JSON metrics file covering: total trade count, clean trade count, rejection count per rule, count of unresolvable P&L records, total realized P&L (USD equivalent).

---

### Deliverable 2: Airflow DAG

Build a production Airflow DAG (`fincore_daily_pipeline.py`).

**Minimum requirements:**

**Source arrival detection (3 independent sensors):**
- One S3KeySensor per source (trades, market_data, portfolio), all running in parallel
- mode=reschedule on all sensors (poke every 10 minutes, timeout after 4 hours)
- If any sensor times out: abort the run, send a PagerDuty/Slack alert naming which source is missing

**Parallel extract after all sensors pass:**
- 3 independent extract tasks (one per source) running in parallel
- A join task (EmptyOperator) that waits for all 3 to complete

**Spark ETL:**
- GlueJobOperator (AWS) or BashOperator for local Spark
- 3 retries with 15-minute delays
- execution_timeout of 2 hours

**Quality gate (BranchPythonOperator):**
- Read quality metrics from S3
- Pass criteria: reject_rate < 2% AND pnl_unresolvable_count < 100
- Fail: send Slack alert with metrics summary and abort
- Pass: proceed to load

**Load TaskGroup:**
- DELETE today's partition from Redshift
- COPY clean trades and portfolio P&L to Redshift

**dbt TaskGroup:**
- dbt run (stg_trades, stg_market_data, fct_daily_trades, agg_portfolio_pnl)
- dbt test (separate task, downstream of dbt run)
- If dbt test fails: alert but do not abort (tests are advisory, not blocking, for this client)

**SLA and alerting:**
- SLA of 6 hours on the final task (pipeline must complete by 6:30 AM if started at 12:30 AM)
- sla_miss_callback sends a PagerDuty webhook
- on_failure_callback sends Slack with task name, date, and log URL for every task

**Idempotency:** Every task must be safely re-runnable for the same date without creating duplicates.

---

### Deliverable 3: dbt Models

Build a complete dbt project with the following models.

**Staging layer:**
- `stg_trades` — from raw COPY of processed trades; rename columns, cast types, coalesce nulls
- `stg_market_data` — from raw COPY of market data; validate price > 0

**Marts layer:**
- `fct_daily_trades` — grain: one row per trade; includes all clean trade fields + realized_pnl
- `dim_instrument` — grain: one row per symbol (built from market data + a seed CSV of static instrument metadata)
- `agg_portfolio_pnl` — grain: one row per portfolio per date; summed P&L, position count, total market value

**dbt tests (schema.yml):**
- `fct_daily_trades`: not_null on order_id, symbol, trade_date, quantity, price; unique on order_id; accepted_values for side (BUY/SELL); accepted_range for quantity (> 0); accepted_range for price (> 0)
- `agg_portfolio_pnl`: not_null on portfolio_id and pnl_date; a custom singular test that verifies total_pnl is the exact sum of fct_daily_trades.realized_pnl for the same portfolio and date
- `stg_market_data`: dbt_utils.accepted_range for close_price (> 0)

**Source freshness:** Define sources in sources.yml with loaded_at_field and a warn_after of 8 hours (data older than 8 hours before the pipeline runs is stale).

---

### Deliverable 4: Monitoring Dashboard Configuration

Produce a CloudWatch Dashboard configuration (JSON) or equivalent monitoring setup that displays:

- Glue job duration trend (last 30 days)
- Record count per run (total, clean, rejected) as a time series
- Rejection rate per run as a percentage metric
- SLA compliance: count of runs completing before 6:30 AM vs after
- dbt test pass rate per run

---

### Deliverable 5: Operations Runbook

Write a 1–2 page `RUNBOOK.md` that the on-call data engineer can use during an incident.

**Required sections:**
- Pipeline overview (architecture diagram in ASCII or link to diagram)
- How to check pipeline status (Airflow UI link, key metrics to look at)
- Common failure scenarios with investigation steps and recovery procedures:
  - Market data sensor timed out (source delayed)
  - Glue job failed with OutOfMemory error
  - dbt test failure on agg_portfolio_pnl custom test
  - Redshift COPY failed with S3 permissions error
- How to re-run a specific date (Airflow clear command)
- Escalation contacts and SLA breach procedure
