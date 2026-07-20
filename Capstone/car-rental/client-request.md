# Project Request: Fleet & Rental Operations Analytics Platform

**From:** Meridian Car Rentals — Group Operations & Technology Office
**To:** AmaliTech Data Engineering Capstone Program
**Subject:** Request for a Data Analytics Platform for Fleet, Booking, and Revenue Operations

---

## 1. About Us

Meridian Car Rentals is a multinational car rental company operating a fleet of several thousand vehicles across branches in North America, Europe, and West Africa. Customers book vehicles through our website, mobile app, and branch counters for both short-term (daily) and extended (weekly/monthly) rentals. We serve leisure travelers, corporate accounts, and airport transfer customers.

Our branches, fleet maintenance teams, finance department, and customer service centers currently operate with limited shared visibility into what is happening across the business in real time. Each region has grown its own spreadsheets and local reports, and there is no single, trusted source of truth that leadership, operations, and finance can all rely on.

## 2. Business Challenges

We are asking for your help because we are experiencing the following problems:

- **Fleet imbalance across locations.** Some branches sit on idle vehicles while others turn away customers due to unavailability, and we have no consolidated way to see this happening as it occurs.
- **Customer dissatisfaction.** We receive recurring complaints about delayed pickups, vehicles not being where they were promised, and confusing or disputed bills.
- **Revenue leakage.** We suspect we are losing money to inconsistent pricing application, uncollected late-return fees, and possibly fraudulent booking activity, but we cannot currently quantify any of this with confidence.
- **Vehicle health and safety blind spots.** Our newer vehicles report live location, speed, and diagnostic data, but nobody is systematically watching this feed for maintenance risks or unsafe driving.
- **Compliance exposure.** We hold sensitive customer and driver information (identity, contact, license, and payment details) and need to be able to demonstrate that this data is handled and governed responsibly.
- **Slow, manual reporting.** Getting a simple answer to "how did we do last week" currently takes days of manual spreadsheet reconciliation across regional teams.

## 3. Project Objectives

We want a modern data platform that:

1. Continuously collects data from our booking systems, our vehicle telematics devices, our fleet management records, our billing systems, and our customer records.
2. Cleans, organizes, and combines this information into a single, trustworthy set of data that the business can query and report from.
3. Applies data quality checks so that we can trust the numbers we see.
4. Runs reliably on a schedule (for the data that arrives periodically) and continuously (for the data that streams in real time), with monitoring so we know if something breaks.
5. Delivers the resulting insight through dashboards that different parts of the business can actually use day to day.
6. Is built and tested in a way that lets us safely move it from a development environment into a live production environment used by staff.

## 4. Success Criteria

We will consider this project successful if it delivers:

- A working end-to-end data pipeline that ingests all five of our identified data sources (see Section 5) without manual intervention once running.
- Data refreshed at a cadence appropriate to each source: near real-time for streaming feeds, and on the stated batch schedule (hourly, daily, or weekly, per source) for file-based feeds.
- A measurable improvement in reporting turnaround: business questions that used to take days to answer should be answerable from a dashboard in minutes.
- At least **two** dashboards (an Executive Dashboard and an Operational Dashboard, described in Section 6) that are refreshed on a schedule matching the underlying data and that business users can navigate without engineering help.
- Documented, automated data quality checks with visible pass/fail results, not just clean-looking output.
- Clear visibility into fleet utilization, booking performance, revenue per vehicle, customer value, late returns, vehicle health/safety alerts, and suspected fraudulent activity — presented as business answers, not raw data dumps.
- A promotion path where the solution is proven in a development environment, verified in a pre-production/testing environment, and then promoted to a production environment without rebuilding it from scratch.
- Documentation sufficient for our own IT team to operate, monitor, and extend the platform after handover.

## 5. Available Data Sources (High-Level)

We can make the following categories of data available to the project team. Exact technical connection details will be provided separately by our IT liaison during project kickoff.

| # | Data Area | How It Arrives | Typical Frequency |
|---|-----------|-----------------|--------------------|
| 1 | **Customer master records** (customer profiles, contact details, license information, loyalty status) | Dropped as files into a shared document library (our internal document management system) | Weekly |
| 2 | **Fleet inventory** (vehicle registry, status, mileage, service schedule) | Delivered as files into a cloud file drop | Hourly |
| 3 | **Billing & payments** (invoices, payment status, payment method) | Delivered as files into a cloud file drop | Daily |
| 4 | **Booking transactions** (reservations, pickups, drop-offs, booking amounts) | Streamed continuously as live events | Real-time, as bookings happen |
| 5 | **Vehicle telematics** (GPS location, speed, fuel level, engine and tire status) | Streamed continuously as live events from onboard devices | Real-time, multiple readings per second per vehicle |

In short: three of our sources arrive as periodic file drops (one through our document management library, two through cloud file storage), and two arrive as continuous live event streams. All sources include some combination of operational and personally identifiable information, and must be handled with appropriate care.

## 6. Required Dashboards

We require **at least two** distinct dashboards, built for two different audiences:

### 6.1 Executive Dashboard

**Audience:** Regional Directors, VP of Operations, Finance leadership.

This dashboard must answer the questions leadership actually asks on a Monday morning: What is our overall fleet utilization, and is it improving or declining? How is revenue performing — booking volume, conversion, and customer value — and how does that compare over time (week over week, month over month)? Which regions or branches are over- or under-performing relative to the rest of the network? A leader should be able to look at this dashboard and understand, in a few minutes, the state of the business and where to focus attention next.

### 6.2 Operational Dashboard

**Audience:** Branch Managers, Fleet/Maintenance Supervisors, Customer Service leads.

This dashboard must answer the questions a branch or fleet manager needs answered to run their day-to-day operation: What is the current state of our fleet — available, rented, in maintenance — and what pickups/drop-offs are coming up? Which returns are late, and which vehicles or bookings need attention right now (telematics-flagged maintenance or safety concerns, payment failures, suspected fraudulent bookings)? Where can a manager drill from a summary view down into the specific branch, vehicle, or booking behind it? The answers should be actionable the same day, not just a historical record.

Teams have full creative freedom in how they design and visualize both dashboards — layout, chart types, tools, and styling are all your call. What is being evaluated is the quality of the insight and how well the dashboard actually answers the business questions above, not visual polish.

## 7. Expected Deliverables

We expect the engagement to produce:

1. An end-to-end data pipeline covering ingestion, transformation, and delivery of all five data sources.
2. Working batch ingestion (for the file-drop sources) and streaming ingestion (for the live event sources).
3. A layered data platform (raw landing → cleaned/validated → business-ready) with clear separation between stages.
4. An automated data quality framework with documented checks and visible results.
5. Automated orchestration and monitoring of the full pipeline, with failure alerting.
6. Three managed environments — development, user acceptance/testing, and production — with a defined, repeatable way to promote changes between them.
7. At least two dashboards as described in Section 6.
8. Documentation covering architecture, data definitions, and operational runbooks, sufficient to hand the solution off to our internal IT team.
9. A final presentation to our stakeholders demonstrating the solution against the success criteria in Section 4.

We look forward to seeing your proposed approach.

— Meridian Car Rentals, Group Operations & Technology Office
