# Snowflake Notes: Topic 1, Why Snowflake

## 1. Problems with traditional (on-prem) warehouses

An on-prem warehouse (Teradata, Oracle Exadata, Netezza) runs on servers your company buys and keeps in its own data center. Storage and compute (CPU/RAM) are bundled in the same machines.

### Scaling
- To add capacity, you buy hardware, wait weeks or months for delivery, install it, and rebalance the data.
- You must size for your *peak* (like month-end), so for most of the month the machines sit idle.
- Scaling down is impossible. You can't return the servers.
- *Example:* a retailer buys servers for Black Friday traffic. They sit 80% idle for the other 11 months, but the company already paid.

### Cost
- Big upfront spending (CapEx) on hardware, licenses, power, cooling, and data-center space.
- Ongoing staff costs for DBAs and hardware engineers.
- You pay whether or not you use it.

### Concurrency
Concurrency means many users or jobs running at the same time. On-prem, everyone shares the same fixed compute.
- *Example:* at 9 AM the finance team runs a huge month-end report, and the ETL load is also running. A manager's small dashboard query now takes 10 minutes because everyone is fighting for the same CPU.
- The classic fix is "run ETL at night, reports in the day," which is rigid and breaks often.

---

## 2. The Hadoop era

Around 2006 to 2015, companies had data too big and too varied for traditional warehouses (logs, clicks, JSON). Hadoop was the answer.

**What Hadoop is:** an open-source framework to store and process huge data across many cheap commodity machines (a "cluster").
- **HDFS** (Hadoop Distributed File System) splits files into blocks and spreads them across machines, with copies for safety.
- **MapReduce** is the processing model: break the job into small pieces, run them on many machines in parallel, then combine the results. Hive later let people write SQL on top.

**Why it was good:** cheap hardware, handles any data type, scales by adding machines.

**Why it hurt:**
- Very hard to set up and maintain (a team of specialists).
- Slow for interactive SQL.
- Storage and compute were still tied together in the same nodes. Adding storage meant adding compute too.
- Small-file problems, complex tuning, painful security.

So companies realized: *we want Hadoop's scale without Hadoop's headache.* That led to cloud data warehouses.

---

## 3. The cloud data warehouse with Snowflake

Snowflake was built from scratch for the cloud (founded 2012). It is a managed service, and you don't install anything.

### a) Separate storage and compute (the key idea)
Snowflake has three layers:
1. **Storage layer:** all data sits in cheap cloud storage (S3 on AWS, Blob on Azure, GCS on Google).
2. **Compute layer:** "virtual warehouses," which are clusters that run your queries.
3. **Cloud services layer:** the brain (security, metadata, query optimization, transactions).

Because storage and compute are separate:
- Many warehouses can read the *same* data at once without slowing each other.
- *Example:* ETL warehouse (Large) loads data, BI warehouse (Small) serves dashboards, Data Science warehouse (Medium) trains models. All hit the same tables with zero interference. This solves the concurrency problem.
- You can grow storage without paying for compute, and the reverse.

### b) Elastic, pay-per-use
- Resize a warehouse (Small to XLarge) in seconds.
- **Auto-suspend:** the warehouse shuts off after, say, 60 seconds of inactivity, and you stop paying for compute.
- **Auto-resume:** it wakes up when a query arrives.
- You pay for compute by the second (minimum 60 seconds each time it starts) and for storage by the TB per month.
- *Example:* month-end report needs power. Scale up to XLarge for 30 minutes, then back down. Cost: 30 minutes of XL instead of owning an XL server all year.

### c) Low admin
"Low admin" does not mean *no* admin. It means the heavy, old-style DBA work is removed:
- No hardware, OS, or software installation or patching.
- No manual indexes, partitions, vacuuming, or statistics collection (Snowflake organizes data automatically through micro-partitions).
- Upgrades happen automatically, with no downtime.
- Backups and recovery come built in (Time Travel, covered in a later topic).

You still need people for: users and roles (security), cost monitoring, data modeling, and pipelines. The DBA role shrinks from "keep the servers alive" to "govern and optimize."

### d) Data sharing
You can share live data with another Snowflake account without copying or sending files.
- *Example:* a company shares a sales dataset with a partner. The partner queries it directly, always sees the latest data, and no ETL or FTP is involved. The data isn't duplicated, so there is no extra storage cost. Snowflake also has a Marketplace where companies publish datasets.

### e) Multi-cloud
Snowflake runs on **AWS, Azure, and Google Cloud**. When you create an account, you **choose one** cloud and region, and the experience (SQL, features) is the same everywhere. That's what "multi-cloud" means.
- You can replicate data across clouds (for example AWS account to Azure account) for disaster recovery or to serve customers on another cloud.
- BigQuery is not something Snowflake connects to as a platform. It is Google's own competing data warehouse.

### f) Semi-structured data support
- **Structured data:** rows and columns with a fixed schema (a customers table).
- **Semi-structured data:** has structure but no fixed table shape: **JSON, Avro, Parquet, ORC, XML.** Think API responses, app logs, event data.
- Old warehouses made you flatten JSON into tables before loading, which was painful. Snowflake lets you load JSON as-is into a column of type **VARIANT** and query it with SQL.
- *Example:* a column holds `{"name":"Ravi","city":"Pune","orders":[...]}` and you query it as `data:city::string`.

---

## 4. Use cases
- **Data warehousing and analytics:** central place for reporting and BI (Power BI, Tableau).
- **Data lake:** store structured and semi-structured data together.
- **Data engineering:** build ELT pipelines (load raw data first, transform inside Snowflake).
- **Data sharing and collaboration:** between teams, partners, customers.
- **Data science/ML:** feed models with large datasets.
- **Applications:** power customer-facing analytics.

---

## 5. Snowflake vs other platforms

Quick definitions:
- **Redshift:** Amazon's data warehouse (AWS only).
- **BigQuery:** Google's data warehouse (Google Cloud only).
- **Synapse:** Microsoft's analytics platform (Azure only).
- **Databricks:** a "lakehouse" platform built on Apache Spark, strongest for big data engineering and machine learning.

| | Snowflake | Redshift | BigQuery | Synapse | Databricks |
|---|---|---|---|---|---|
| Owner | Independent | Amazon | Google | Microsoft | Independent |
| Clouds | AWS, Azure, GCP | AWS only | GCP only | Azure only | AWS, Azure, GCP |
| Storage/compute split | Fully separate | Partly (newer RA3 nodes) | Fully separate | Partly | Separate (data in your cloud storage) |
| Pricing style | Per-second compute + storage | Per node-hour (or serverless) | Per query scanned or slots | Per resource used | Per compute unit (DBU) + cloud cost |
| Admin effort | Low | Medium (tuning, vacuum) | Very low | Medium | Medium |
| Best at | SQL analytics, sharing, ease | Teams deep in AWS | Serverless, Google ecosystem | Microsoft shops | Spark, ML, heavy engineering |
| Main language | SQL | SQL | SQL | SQL (+Spark) | Python, Scala, SQL |

**Simple way to remember:**
- Snowflake = easiest, SQL-first, works on any cloud.
- Redshift / BigQuery / Synapse = each is the "home" warehouse of one cloud vendor.
- Databricks = data engineering and AI first, SQL second (though the gap is closing).

---

## 6. Hands-on: Snowflake vs BigQuery

| | Snowflake | BigQuery |
|---|---|---|
| Owner | Independent company | Google |
| Clouds | AWS, Azure, GCP | GCP only |
| Compute | You create **virtual warehouses** and choose their size | **Serverless.** You never create or size compute |
| Pricing | Pay for warehouse running time (credits) + storage | Pay per **TB of data scanned** by each query (or reserve "slots") + storage |
| Semi-structured data | **VARIANT** type holds any JSON, Parquet, Avro, etc. | No VARIANT. Has a **JSON** type plus **STRUCT/ARRAY**, which need more schema planning |
| Concurrency | Separate warehouses per team, so no interference | Google shares a big pool of compute automatically |
| Control | More control (size, auto-suspend, multi-cluster) | Less control, less to manage |
| Data sharing | Live sharing + Marketplace | Analytics Hub, mostly within Google |
| Best for | Multi-cloud, predictable per-team costs | Teams already on Google Cloud, spiky ad-hoc queries |

**Key idea:** Snowflake charges for **time the compute is on**. BigQuery charges for **how much data you read**. A careless `SELECT *` on a huge table is expensive in BigQuery. A warehouse left running is expensive in Snowflake.

---

## 7. Hands-on: Cost scenario

Illustrative numbers only. Real prices vary by edition and region, so check Snowflake's pricing page before quoting them.

**Assumptions**
- A Medium warehouse uses **4 credits per hour**.
- 1 credit costs about **$3**.
- So Medium costs **$12 per hour**.

**Option A: always on** (like an on-prem server)
- 24 hours x 30 days = 720 hours
- 720 x $12 = **$8,640 per month**

**Option B: runs 2 hours a day** (auto-suspend handles the rest)
- 2 hours x 30 days = 60 hours
- 60 x $12 = **$720 per month**

**Result:** saves **$7,920 per month, about 92%**, for the same work. That is what "elastic, pay-per-use" means in money.

**Bonus:** if month-end needs more power, scale up for just that window. XLarge uses 16 credits/hour, so 1 hour costs about $48, then scale back down.

---

## 8. Hands-on: Free trial sign-up
Already done: Snowflake free trial account created.

---

## Quick recap
1. On-prem: hard to scale, expensive, concurrency clashes.
2. Hadoop: cheap and scalable, but complex and still storage/compute-coupled.
3. Snowflake: separated storage and compute, elastic, low admin, data sharing, multi-cloud, native JSON.
4. Snowflake = multi-cloud, you size compute, pay for time. BigQuery = Google only, serverless, pay for data scanned.
5. Always-on $8,640 vs 2 hours/day $720.
