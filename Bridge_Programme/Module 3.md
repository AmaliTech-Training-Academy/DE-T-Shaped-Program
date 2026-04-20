# MODULE 3: Cloud Literacy — How Infrastructure Works

## Why This Module Exists

Data engineering happens in the cloud. Not sometimes — almost always. When you join a data engineering project, the data lives in S3 or Azure Data Lake, the transformations run on cloud compute, and the pipelines are orchestrated by cloud-hosted tools. You cannot contribute to these projects without a working mental model of how cloud infrastructure works.

This module does not teach you to build cloud infrastructure. It teaches you to understand it well enough to work within it.

---

## Module 3 Glossary

**Cloud provider** — A company that rents computing resources (storage, compute, networking, services) over the internet. The three major providers are: AWS (Amazon Web Services), Microsoft Azure, and Google Cloud Platform (GCP).

**AWS (Amazon Web Services)** — Amazon's cloud platform. The largest and most widely used cloud provider. Skills 4, 5, 8, 9, 11 in this library use AWS.

**Microsoft Azure / Microsoft Fabric** — Microsoft's cloud platform. Skills 6 and 7 use Fabric (Microsoft's data platform, built on Azure).

**Region** — A physical location (data centre cluster) where a cloud provider operates. AWS has regions like `us-east-1` (Virginia), `eu-west-1` (Ireland). Data that you store in a region physically lives in that location.

**Storage** — A place to keep data when a computer is turned off. In the cloud, the main storage service is object storage (S3 on AWS, ADLS on Azure). Think of it as a hard drive in the internet.

**Compute** — The CPU and memory that runs your code. In the cloud, compute is rented on-demand. You pay for it while your code runs and stop paying when it finishes.

**S3 (Simple Storage Service)** — AWS's object storage service. You store files (CSV, Parquet, JSON, images, anything) in "buckets." Data lakes are almost always built on S3.

**Bucket** — A container in S3 that holds files. Similar to a folder on your computer, but accessible over the internet via a URL.

**IAM (Identity and Access Management)** — AWS's system for controlling who can access what. Every person, application, and service needs an IAM identity with specified permissions to access AWS resources.

**Lambda** — AWS's serverless compute service. You upload a function (Python, Node.js, etc.), and AWS runs it when triggered. You pay only for the milliseconds it runs. No server to manage.

**EC2** — AWS's virtual machine service. You rent a computer in AWS's data centre, choose its size (CPU, memory), and run it continuously. More control than Lambda but more to manage.

**API (Application Programming Interface)** — A way for two software systems to communicate. An API defines: "Send me a request in this format, I will respond with data in this format." When a data engineer says "pull data from the API," they mean: write code that sends HTTP requests to an endpoint and receives JSON data back.

**HTTP** — The protocol (set of rules) that web communication uses. GET requests retrieve data. POST requests send data.

**JSON** — The most common format for data sent by APIs. Looks like a Python dictionary: `{"name": "Alice", "age": 30}`.

**VPC (Virtual Private Cloud)** — A private, isolated network within AWS. Resources inside a VPC can communicate with each other privately, without going over the public internet.

**RDS** — AWS's managed relational database service. AWS runs and maintains the database (PostgreSQL, MySQL, etc.) for you — you just connect and query.

**OneLake** — Microsoft Fabric's unified storage layer. All data in Fabric (Lakehouses, Warehouses, KQL databases) is stored in OneLake — one storage location for everything.

**Managed service** — A cloud service where the provider handles the operational complexity (backups, scaling, patching, failover). You use it; they run it. DynamoDB, S3, and Lambda are all managed services.

---

## Concept Explainer 3.1: The Cloud as Rented Infrastructure

Before cloud computing, every company that needed servers had to buy them. Physical machines that sat in a room called a data centre. They needed to be powered, cooled, maintained, and replaced when they broke. When a company grew, buying more servers took months. When traffic was low, the servers sat idle and wasted money.

Cloud computing changed this by making infrastructure rentable by the hour (or second).

**The analogy:** Think of cloud computing like a rental car company vs buying a car.
- **Buying a car (on-premise):** High upfront cost. You own it. You maintain it. When it breaks, it is your problem. It sits in your garage unused at night.
- **Renting a car (cloud):** Pay only when you use it. Different size for different trips. Someone else handles maintenance. Return it when you are done.

Cloud providers offer three fundamental things:

| Resource | What It Is | Cloud Example |
|---|---|---|
| **Storage** | Space to keep data | S3 (AWS), Azure Data Lake |
| **Compute** | CPU + memory to run code | EC2, Lambda, EMR (AWS) |
| **Networking** | How things connect to each other | VPC, load balancers |

On top of these three fundamentals, they offer **managed services** — databases, machine learning platforms, streaming systems, and more, all operated by the cloud provider.

---

## Concept Explainer 3.2: Object Storage — The Foundation of the Data Lake

The most important cloud concept for data engineers is **object storage**. This is where all data lakes live.

**How object storage works:**
- You store files ("objects") in containers ("buckets")
- Each file has a path (like a URL): `s3://my-company-datalake/bronze/orders/2026-03-20.parquet`
- Files can be any type and any size
- There are no folders (it only looks like folders — the `/` in the path is part of the filename)
- Anyone with the right permissions can access any file from any computer in the world
- Storage is cheap: approximately $0.023 per GB per month on AWS S3

**Why data lakes use object storage:**
- Unlimited scale (store petabytes without planning ahead)
- Cheap (compared to databases)
- Accessible by any compute engine (Spark, Athena, Python scripts — all can read from S3)
- Durable (AWS S3 stores 3 copies of every file in different physical locations)

**The path convention in data lakes:**
```
s3://bucket-name/
├── bronze/
│   └── orders/
│       └── date=2026-03-20/
│           └── orders_20260320.parquet
├── silver/
│   └── orders/
│       └── date=2026-03-20/
│           └── orders_clean.parquet
└── gold/
    └── fact_orders/
        └── fact_orders.parquet
```

The `bronze/silver/gold` structure maps directly to what you will learn in SK-08 (Lakehouse Architecture).

---

## Concept Explainer 3.3: APIs — How Data Engineers Get Data from External Systems

When a company wants to give other systems access to its data, it builds an API. An API is like a waiter in a restaurant: you do not go into the kitchen yourself; you give your order to the waiter, the waiter brings back what you asked for.

As a data engineer, you will frequently write code that calls APIs to pull data into your pipeline. The most common type is a **REST API over HTTP**.

**How it works:**
1. You send an HTTP request to a URL (called an endpoint)
2. The server processes the request and returns data (usually in JSON format)
3. Your code parses the JSON and puts it into a DataFrame or writes it to a file

```python
import requests
import pandas as pd

# This is what calling an API looks like in Python

# Make a GET request (asking for data)
response = requests.get(
    url="https://api.somecompany.com/v1/orders",
    headers={"Authorization": "Bearer your-api-key-here"},
    params={"start_date": "2026-01-01", "limit": 1000}
)

# Check if the request succeeded (200 means OK)
if response.status_code == 200:
    data = response.json()          # Parse the JSON response
    df = pd.DataFrame(data["orders"])  # Convert to DataFrame
    print(f"Loaded {len(df)} orders")
else:
    print(f"Request failed: {response.status_code}")
    print(response.text)

# Many APIs use pagination — they return a limited number of results at a time
# and give you a "next page" URL or cursor to get the rest
```

**Pagination** is very common. An API might say "I will return 1,000 records per request. If there are more, here is how to get the next page." Data engineers write loops that keep calling the API until all pages are collected.

---

## Concept Explainer 3.4: Serverless — Lambda and "No Server to Manage"

"Serverless" sounds paradoxical. There are servers. You just do not manage them.

**Traditional compute:** You rent a virtual machine (EC2). It runs 24/7. You pay whether your code is running or not. You are responsible for the operating system, patches, and restarts.

**Serverless compute (Lambda):** You upload a function. AWS runs it when triggered (an API call, a file arriving in S3, a timer). You pay only for the milliseconds it runs. Nothing to patch, nothing to restart, no idle cost.

**The data engineering use cases for Lambda:**
- Process a file when it arrives in S3 (S3 event trigger → Lambda → parse file → write to database)
- Call an API on a schedule (EventBridge timer → Lambda → call API → write results to S3)
- Process events from a stream (DynamoDB Streams → Lambda → route events → update search index)
- Light transformation that does not need a full Spark cluster

Lambda is limited: it can only run for 15 minutes maximum, and it only has modest memory (up to 10GB). For large-scale transformations (billions of rows), you need a different compute service (Spark on EMR, Glue, Fabric Spark). Lambda is for small, event-driven workloads.

---

## Concept Explainer 3.5: IAM — Who Can Access What

Every resource in AWS has an owner and a set of access rules. IAM is the system that enforces this.

**The three concepts you must understand:**

**User** — A person or application identity. A developer has an IAM user. A Lambda function has an IAM role.

**Policy** — A JSON document that says "this identity IS or IS NOT allowed to perform these actions on these resources." Example: "Can read from S3 bucket X. Cannot delete anything."

**Role** — Like a User, but meant to be assumed temporarily by a service or application. When your Lambda function needs to write to DynamoDB, it assumes an IAM role that grants that permission.

```json
// Example IAM policy: Allow reading from one specific S3 bucket
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-company-datalake",
                "arn:aws:s3:::my-company-datalake/*"
            ]
        }
    ]
}
```

The principle of **least privilege**: every identity should have the minimum permissions needed to do its job, nothing more. A Lambda that reads files from S3 should not have permission to delete files or access other buckets. This is a security principle you will encounter in SK-04 and SK-09.

---

## Module 3 Self-Check

1. What is the difference between storage and compute in a cloud context? Give a real AWS example of each.

2. You want to run a Python script that processes a CSV file every morning at 6am, takes about 5 minutes to run, and is triggered automatically. Would you use EC2 or Lambda? Why?

3. You have data in `s3://company-lake/bronze/sales/2026-03.csv`. What is the bucket name? What is the key (path) of the file?

4. An API endpoint returns a 401 status code. What does that usually mean and what do you check?

5. Your Lambda function writes processed results to a DynamoDB table. The function keeps getting "Access Denied" errors. Where in AWS do you go to fix this?

---

## Module 3 Resources

### Read First (Free, Currently Live)
1. **AWS Cloud Practitioner Essentials — Free Digital Course** (explore.skillbuilder.aws) — AWS's own free introduction to cloud concepts. The "Cloud Concepts" and "AWS Core Services" modules are directly relevant. Takes about 6 hours total. You do not need to do the whole thing — focus on the Storage, Compute, and Security sections.

2. **What is a REST API? — Red Hat** (redhat.com/en/topics/api/what-is-a-rest-api) — A clear, jargon-light explanation of what REST APIs are, how HTTP methods work (GET, POST, PUT, DELETE), and why APIs matter for integration. 10-minute read.

3. **AWS IAM Getting Started Guide** (docs.aws.amazon.com/IAM/latest/UserGuide/getting-started.html) — The official introduction to IAM. Read the "Understanding How IAM Works" section. The policy examples are the most valuable part.

### Watch (Free, Currently Live)
1. **AWS Basics For Beginners — freeCodeCamp** — Search "freeCodeCamp AWS cloud practitioner" on YouTube. freeCodeCamp published a full free AWS Cloud Practitioner course. Watch the first 2 hours covering cloud concepts, S3, EC2, and IAM.

2. **What is an API? — MuleSoft** — Search "MuleSoft what is an API" on YouTube. A 3-minute animated explanation that is genuinely the clearest introduction to APIs available for free.

### Practice (Free)
1. **AWS Free Tier** (aws.amazon.com/free) — Create a free AWS account. Every learner should do this. The free tier gives you 12 months of limited access to most AWS services at no cost. Create an S3 bucket, upload a file, and make it private. This takes 20 minutes and makes everything concrete.

