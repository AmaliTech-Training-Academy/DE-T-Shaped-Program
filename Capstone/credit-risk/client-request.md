# Data & Analytics Platform Request — Meridian Trust Bank

**From:** Office of the Chief Risk Officer, Meridian Trust Bank
**To:** AmaliTech Data Engineering Team
**Subject:** Request for a Credit Risk & Portfolio Analytics Platform
**Date:** 11 July 2026

## 1. Business Background

Meridian Trust Bank is a retail bank serving individual and small-business borrowers across personal loans, auto loans, mortgages, business loans, and student loans. Our loan book has grown quickly over the past two years, and our reporting has not kept pace with it.

Today, our view of portfolio risk is built from a patchwork of end-of-day spreadsheet exports, manually stitched together by our finance team, refreshed on no fixed schedule, and often out of date by the time leadership reviews them. Our risk officers have no way to see delinquency or suspicious account activity as it happens — they find out about problems days after they occur. Regulators are also asking harder questions of us (capital adequacy, non-performing loan exposure, and portfolio quality reporting in line with Basel III expectations), and we currently cannot answer them quickly or with confidence in the numbers.

We need a proper, governed data platform behind our risk reporting — not another one-off spreadsheet.

## 2. Current Business Challenges

- **Siloed data.** Loan origination, repayment, external credit bureau data, customer account activity, and market/economic indicators all live in different systems that don't talk to each other.
- **Delayed visibility.** Our risk team reviews yesterday's (or last week's) numbers. There is no live picture of delinquency trending or unusual transaction activity.
- **Manual, error-prone reporting.** Analysts rebuild the same pivot tables every week by hand, which is slow and inconsistent from one analyst to the next.
- **No single source of truth for leadership.** Executives, risk officers, and analysts each look at different numbers and disagree on what "the default rate" or "the exposure" actually is.
- **Regulatory pressure.** We need repeatable, auditable reporting on portfolio quality and non-performing exposure, and we cannot currently produce it on demand.
- **Growing data volume.** Transaction and market activity are only going to increase in volume and velocity, and our current manual process will not scale.

## 3. Project Objectives

We are asking AmaliTech to design and build an end-to-end data platform that:

1. Ingests our recurring batch data (loan application submissions, loan repayment records, and periodic external credit bureau reports) on a reliable, automated schedule.
2. Ingests our real-time data (customer account transaction activity and market/economic indicator feeds) as continuous event streams, so risk-relevant activity is visible with minimal delay.
3. Cleans, validates, and organizes all of this data into a well-governed, layered data store that both technical and non-technical staff can trust.
4. Builds an automated data quality checking process so we know when something is wrong with incoming data before it reaches a dashboard.
5. Delivers business-ready dashboards for two very different audiences: our executive leadership team and our front-line risk & operations team.
6. Is built and promoted through proper development, testing, and production environments, the way any production banking system should be.

## 4. Success Criteria

The engagement will be considered successful when:

- Daily batch files (loan applications, loan repayments) are ingested and reflected in the platform within **4 hours** of file arrival, with zero manual intervention.
- The weekly external credit bureau report is ingested and reflected in the platform within **1 business day** of arrival.
- Customer transaction activity and market data events are visible in the operational dashboard within **15 minutes** of occurring.
- At least **98% of ingested records** pass automated data quality checks (completeness, valid ranges, referential consistency between customers, loans, and repayments); records that fail are logged and quarantined rather than silently dropped or silently accepted.
- Executive leadership can view portfolio-wide trends (defaults, repayment performance, portfolio profitability, non-performing exposure) for any trailing period without asking IT for a custom export.
- Risk officers can drill from a portfolio-level view down to the individual flagged loan or customer without leaving the dashboard.
- The solution has been demonstrated running in a development environment, validated in a test/UAT environment, and promoted into a production environment following a repeatable release process.
- All of the above is documented well enough that a new analyst could understand where the numbers come from without asking the original build team.

## 5. Available Data

We can make the following data available. Exact delivery mechanics can be worked out with our IT team, but broadly:

| Data | Nature | How it arrives |
|---|---|---|
| **Loan Applications** | New loan applications submitted by customers — applicant details, loan type, amount, term, interest rate, and approval outcome | Daily batch file drop, delivered to a cloud file location |
| **Loan Repayments** | Scheduled and actual repayment activity against active loans — due dates, payment dates, amounts paid, payment status | Daily batch file drop, delivered to a cloud file location |
| **Credit Bureau Reports** | Third-party credit bureau data on our customers — credit score, open loan count, prior default history, recent credit inquiries | Weekly report file, delivered to a shared document library (our compliance team's standard channel for receiving vendor reports) |
| **Customer Transaction Activity** | Live account activity — purchases, transfers, withdrawals, deposits, fees, and interest postings on customer accounts | Continuous real-time event stream |
| **Market & Economic Indicators** | Relevant market indices and interest-rate benchmarks that influence portfolio risk and pricing | Frequent real-time/interval event feed |

We expect the batch file drops and the shared document library to be treated as standard cloud file-landing channels, and the transaction/market feeds to be treated as standard streaming event sources. We do not have a preference on the specific tools used to receive them, as long as the pipeline is reliable and auditable.

Note: some of this data includes customer personally identifiable information (name, contact details, date of birth, government ID number, account number). We expect this to be handled with appropriate access controls throughout the platform.

## 6. Required Dashboards

We require **at least two** dashboards, built for two different audiences:

### Dashboard 1 — Executive Dashboard

Audience: Chief Risk Officer, executive leadership, board reporting.

This dashboard needs to answer, for a leadership audience: What is our current default rate, and is it trending up or down? How is portfolio profitability/yield tracking over time? Where does our non-performing loan exposure and early delinquency stand today, and how has it moved month-over-month or quarter-over-quarter? How is lending performance holding up across our different loan types and portfolio segments? Leadership should be able to walk into a board-level review and get these answers at a glance, without needing a custom export from IT.

### Dashboard 2 — Risk & Operations Dashboard

Audience: risk officers and loan operations staff doing day-to-day monitoring.

This dashboard needs to answer, for a front-line risk/operations audience: Where is our non-performing exposure and delinquency concentrated (by loan type, segment, region, or individual account)? Which loans or customers need attention right now — missed payments, unusual transaction activity, deteriorating credit profile — and why? What should a risk officer follow up on next, and can they get from a portfolio-level view down to the specific loan or customer driving the concern without leaving the dashboard?

We are open to discussing the exact metrics and layout used to answer these questions on each dashboard — teams have full creative freedom in design and visualization choices. We will be evaluating the quality, clarity, and correctness of the insight and answers each dashboard delivers against the business questions above, not visual polish.

## 7. Expected Deliverables

By the end of this engagement, we expect:

- A complete, end-to-end data pipeline covering all five data sources described above.
- Working batch ingestion for the batch/file-based sources and working streaming ingestion for the real-time sources.
- A layered data architecture (raw landing, cleaned/conformed, and business-ready layers) rather than a single flat dump of data.
- An automated data quality framework with documented rules and a way to see what passed/failed.
- Automated orchestration and scheduling of the full pipeline, with basic monitoring.
- The solution deployed and demonstrated across separate development, test/UAT, and production environments, promoted through a repeatable process.
- The two dashboards described above (Executive and Risk & Operations), built on top of the business-ready data layer.
- Documentation covering the architecture, the data model, the data quality rules, and how to operate the pipeline.
- A final walkthrough/presentation of the solution to our team.

We look forward to seeing your proposed architecture and project plan.

— Office of the Chief Risk Officer, Meridian Trust Bank
