# SK-04 CAPSTONE PROJECT: AWS Data Platform for SwiftHaul Logistics

---

## PROJECT OVERVIEW

### The Business Scenario

You have been deployed to **SwiftHaul**, a logistics company operating a fleet of **10,000 trucks** across North America. Every truck emits a telemetry payload every 10 seconds: GPS coordinates, speed, fuel level, engine temperature, odometer reading, and four diagnostic trouble codes (DTCs).

That is **1,000 telemetry events per second** — 86.4 million events per day — generating approximately **50GB of raw data per day** and **1.5TB per month**.

SwiftHaul's current architecture is a single MySQL RDS instance that receives all telemetry via a custom API. The DBA team reports:

- **Storage at 3TB capacity** — will hit the 4TB ceiling in 6 weeks
- **Query performance degraded** — the operations dashboard (which queries the last 30 days of GPS data for 10,000 trucks) takes 8 minutes to load
- **No historical analysis** — the MySQL instance only retains 90 days of data; insurance claims require 2-year lookbacks that are currently impossible
- **No real-time alerting** — the team learns about engine failures from truck drivers calling dispatch, not from automated alerts

The Head of Operations wants a platform that can: ingest telemetry in real time, store 5 years of history, power a sub-5-second operations dashboard, and trigger alerts when a truck's engine temperature exceeds a threshold or fuel drops below 10%.



---

## PROJECT REQUIREMENTS

### Deliverable 1: Architecture Diagram

Produce a complete architecture diagram (ASCII, draw.io, or Mermaid) showing:

- All AWS services used with their names and types
- Data flow arrows between services (label with approximate throughput or volume)
- Security boundaries (VPC, public vs private subnets)
- Network paths (which traffic goes through VPC endpoints, which uses internet)
- At least one high-availability consideration explicitly called out

The diagram must be accompanied by a 1-page written narrative explaining: (a) why you chose each service over its alternatives, (b) how the architecture handles the real-time alerting requirement alongside the batch analytics requirement.

---

### Deliverable 2: S3 Data Lake Configuration

**Bucket design and configuration (AWS CLI or CloudFormation):**
- Create the bucket with versioning, Block Public Access, and SSE-KMS encryption using a CMK
- Create a KMS CMK with a key policy that allows: the Firehose role to GenerateDataKey, the Glue ETL role to Decrypt/GenerateDataKey, and the Redshift role to Decrypt
- Apply a lifecycle policy with transitions justified for the telemetry access pattern (telemetry from 2 days ago is accessed frequently for operations; 6-month-old data is accessed rarely for insurance claims; 5-year-old data is almost never accessed)

**Partitioning strategy:**
- Propose and justify the primary partition scheme for raw telemetry (choose between `truck_id/date`, `date/truck_id`, `date/hour`, or another scheme)
- State which 2 of the 3 common query patterns below are well-served by your primary partition scheme, and propose a secondary materialized copy that serves the third:
  - Pattern A: "Show me all events for truck TRK-0042 for the last 7 days"
  - Pattern B: "Show me all trucks with engine temperature > 200°F in the last hour" (operations dashboard)
  - Pattern C: "Show me all events for yesterday's date across all trucks" (nightly ETL batch)

---

### Deliverable 3: Kinesis Ingestion Pipeline

**Shard sizing calculation:**
- Show your work: calculate required shards for 1,000 events/sec at 600B average size in PROVISIONED mode
- Calculate the monthly cost for PROVISIONED vs ON\_DEMAND at this throughput
- State your recommendation with justification (consider: the fleet telemetry is highly predictable — trucks are active 6 AM–10 PM local time; almost nothing at night)

**Data Stream configuration (boto3 or CloudFormation):**
- Create the stream with your chosen mode
- Justify the partition key choice (truck\_id? fleet\_region? random?)

**Firehose delivery stream:**
- Buffer size and interval justified for the telemetry use case (target file size reasoning)
- Format conversion to Parquet using a Glue Catalog table
- Error output prefix configured
- CloudWatch alarm on DeliveryToS3.DataFreshness > 600 seconds

**Real-time alerting consumer (Lambda):**
- Write a Lambda function that is triggered by the Kinesis stream (event source mapping)
- The Lambda should: parse each telemetry record, check if `engine_temp_celsius > 95` OR `fuel_level_pct < 10`, and if either threshold is breached, publish an alert to an SNS topic with the truck ID, timestamp, and which threshold was exceeded
- Handle partial batch failures (use `bisect_on_function_error: true` in the event source mapping to isolate and retry failed records)

---

### Deliverable 4: Redshift Data Warehouse

**Table DDL with full design justification:**

Design DDL for the following tables. For each table, document:
- Distribution style and key (or ALL) — and WHY
- Sort key (compound or interleaved) — and WHY
- Compression encoding for each column — and WHY (be specific: why BYTEDICT for `engine_status` and not ZSTD?)

Tables to design:
- `fact_telemetry` — the main fact table (1B rows/year, joins on truck\_id)
- `dim_truck` — 10,000 trucks with make, model, year, fleet, assigned driver
- `dim_driver` — 12,000 drivers with name, license class, region, hire date
- `dim_route` — assigned routes per truck per day (~50K rows/month)
- `dim_date` — standard date dimension

**External schema:**
- Create a Spectrum external schema over the S3 historical telemetry
- Write one Spectrum query that answers: "For each truck, what was the average fuel consumption rate (fuel drop per km) in Q1 2025?"

**Materialized view:**
- Create a materialized view `mv_hourly_fleet_summary` that pre-aggregates: avg/max engine temp, avg speed, total km driven, and alert count per truck per hour
- Explain the refresh strategy (auto vs scheduled) and its implications for the operations dashboard

---

### Deliverable 5: IAM Security Model

Design and implement a 5-role least-privilege security model:

| Role | Assumed By | Minimum Required Permissions |
|---|---|---|
| `swifthaul-truck-producer-role` | Truck IoT devices (EC2 or IoT Core) | Write to Kinesis stream only |
| `swifthaul-lambda-consumer-role` | Lambda alerting function | Read from Kinesis; publish to SNS; write CloudWatch logs |
| `swifthaul-firehose-delivery-role` | Kinesis Firehose | Read Kinesis; write S3 (raw only); read Glue table schema; encrypt via KMS |
| `swifthaul-glue-etl-role` | Glue ETL jobs | Read S3 raw; write S3 processed/curated; update Glue Catalog; CloudWatch logs; KMS |
| `swifthaul-redshift-role` | Redshift cluster | Read S3 data lake (read-only); read Glue Catalog (read-only); KMS decrypt |

For each role: write the complete IAM policy JSON, test it using the IAM Policy Simulator (screenshot or CLI output), and verify that the role CANNOT perform at least one action it should not be able to do (e.g., Firehose cannot write to Redshift; Lambda cannot delete from S3).

---

### Deliverable 6: Detailed Cost Estimate

Produce a monthly cost breakdown for the following scenarios:

**Scenario A: Current scale (10K trucks, 1,000 events/sec)**
Break down costs by service. Identify the largest cost driver.

**Scenario B: 3x growth (30K trucks, 3,000 events/sec)**
Identify which services scale linearly with volume, which are roughly fixed, and which have step-function cost increases (e.g., needing to add a Redshift node).

**Scenario C: Cost optimisation proposal**
Identify one change to each of the 3 most expensive services that would reduce cost without degrading the SLA. Calculate the savings for each. Total savings must be at least 20% vs Scenario A.

Use the AWS Pricing Calculator and show your work. Acknowledge any assumptions made (e.g., data transfer within region is free).

---

### Deliverable 7: Scaling Plan

Write a 1–2 page scaling plan addressing: "SwiftHaul acquires a competitor and grows from 10K to 100K trucks overnight. How does the architecture handle this without re-architecture?"

Address each service:
- **Kinesis:** What changes are needed (if any) for 10x throughput?
- **Firehose:** Does it require any changes?
- **S3:** Does it require any changes? Any cost cliffs?
- **Lambda alerting consumer:** What happens to concurrency? Will it scale automatically?
- **Redshift:** Can the current cluster handle 10x data volume? What is the scaling path?
- **Glue ETL:** Are there any bottlenecks at 10x volume?

Identify the **single most likely bottleneck** in the current design under 10x load and propose a specific mitigation.

