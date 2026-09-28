# Phase 1 | Topic 1: What Snowflake Is

## Chunk 1: The Definition

**Snowflake is a cloud-native data platform, delivered as SaaS.**

| Term | Meaning |
|---|---|
| **Cloud-native** | Built from the ground up to run on the cloud. Not an old database moved to the cloud later. Its design (e.g., separating storage from compute) only makes sense in the cloud. |
| **SaaS (Software as a Service)** | You rent a finished service instead of installing it. No servers to buy, no software to install, no patching or upgrades. Snowflake handles all of that. You pay for what you use. |
| **Data platform** | More than a warehouse. Covers data warehousing, data lakes, data engineering pipelines, data sharing and applications, all in one place. SQL is the main language. |

- Runs on top of **AWS, Azure, or Google Cloud** (chosen when creating an account).
- **Analogy:** using electricity from the grid instead of running your own generator.

**Related terms**
- **Data warehouse:** stores structured data that has been cleaned and organized for reporting and analysis.
- **Data lake:** stores any kind of data (structured, semi-structured, unstructured) in large volumes, usually in raw form.
- **Data pipeline:** an automated, repeatable flow of data: ingest -> transform -> load -> serve to users.

**Interview angle:** *"Is Snowflake a database, a data warehouse, or something else?"*
It began as a cloud data warehouse, but today it is positioned as a full data platform: it also covers data lakes, pipelines, sharing, and app building.

---

## Chunk 2: Data Sharing and Applications

### Data Sharing
Giving another Snowflake account access to your data **without copying it or sending files**.

- **Old way:** export a CSV, email it, or send it via SFTP. The receiver gets a copy that goes stale as soon as your data changes, and now two copies must be managed.
- **Snowflake way:** the **provider** (data owner) shares the data, and the **consumer** (receiver) queries it directly. The consumer always sees **live, up-to-date data**. Nothing is physically copied or moved.
- The provider only **grants access**. It stays in control and can **revoke** access at any time.
- **Analogy:** instead of photocopying a book and mailing it, you give someone a library card to read the original.

### Applications
- Companies can package code + data into an app, and other organizations install it in their own Snowflake account. Their data stays inside their own account.
- Snowflake also lets you build simple dashboards/apps directly on your data using **Streamlit**.
- Think of it as an app store inside the data platform.

**Interview angle:** *"How is Snowflake data sharing different from exporting files?"*
Live access, no copies, no stale data, and no extra storage cost for the consumer.

---

## Chunk 3: Core Characteristics of Snowflake

1. **Separation of storage and compute:** each scales independently (see recall section below).
2. **Elasticity:** resize or add compute in seconds, and shut it down when idle.
3. **Near-zero administration:** no hardware, no index tuning, no manual partitioning, no vacuuming. *(Vacuuming = routine cleanup of old data that some databases require you to run yourself.)*
4. **Multi-cloud:** the same Snowflake experience on AWS, Azure, and Google Cloud.
5. **Pay-per-use:** storage billed by volume; compute billed only while a warehouse is running.
6. **Multiple data types:** structured (tables) and semi-structured (JSON, Parquet) in one platform, queried with SQL.
7. **Built-in data sharing and security:** encryption on by default, access controlled through roles.

**Interview angle:** *"Why is Snowflake called low-maintenance?"*
Tuning tasks like indexing, partitioning, and storage management are automated by the cloud services layer.

---

## Recall: Storage vs Compute Separation

**Old way:** in traditional databases, storage and compute were bundled on the same machines. To get more of one, you had to buy both, and many users querying at once competed for the same resources.

**Snowflake's three independent layers**
| Layer | Role |
|---|---|
| **Storage** | Your data, kept in compressed chunks called **micro-partitions** on the cloud provider's storage |
| **Compute** | **Virtual warehouses**: the "workers" that run queries. Many can run at once, in different sizes, on the same data |
| **Cloud services** | The "manager": security, query planning, metadata |

**Why it matters**
- Compute scales up or down without touching the data.
- Different teams use separate warehouses on the same data, so nobody slows anyone else down.
- Storage and compute are billed separately; compute only while running.

**Scale up vs scale out**
- **Scale up** (bigger warehouse): makes one heavy query faster.
- **Scale out** (more clusters): serves more users at the same time.

---

## Glossary
| Term | Meaning |
|---|---|
| Cloud-native | Designed from the start to run on the cloud |
| SaaS | Renting a fully managed service instead of installing software |
| Data platform | One system covering warehousing, lakes, pipelines, sharing, apps |
| Data warehouse | Cleaned, structured data organized for analysis |
| Data lake | Large-volume storage of raw data of any type |
| Pipeline | Automated flow: ingest -> transform -> load -> serve |
| Provider / Consumer | The account that shares data / the account that receives access |
| Streamlit | Tool for building simple data apps and dashboards inside Snowflake |
| Vacuuming | Routine cleanup of old data in some databases |
| Micro-partition | Small automatic chunk of table data in Snowflake storage |
| Virtual warehouse | Compute cluster that runs queries |

---
**Next Topic:** Problems Snowflake solves
