# Phase 1 | Topic 2: Problems Snowflake Solves

## 1. Big Upfront Cost & Capacity Guessing
On-prem warehouses need servers bought in advance based on a **capacity** guess.
- Guess too small -> crashes/slowdowns during peak load
- Guess too big -> paying for idle capacity all year
- Scaling up later takes weeks/months (ordering, installing hardware)

**Snowflake:** rent compute in the cloud, scale in seconds, pay only for use.

## 2. Storage & Compute Tied Together
- **Storage** = where data sits. **Compute** = CPU/memory that runs queries.
- Traditional systems bundle both on the same servers -> shared, limited power.
- **Concurrency** (many users querying at once) causes **resource contention** (teams slow each other down). E.g., finance's heavy report freezes marketing's dashboard.

**Snowflake:** separates storage and compute. Each team gets its own **virtual warehouse** on the same data — no contention. Compute and storage scale independently.

## 3. Heavy Maintenance
DBAs had to manually handle:
- **Indexing** (speeds up lookups, like a book's index)
- **Partitioning** (splitting big tables into smaller pieces)
- **Tuning**, **backups**, **patching/upgrading** (often with downtime)

**Snowflake:** automates all of this — auto-partitions data into **micro-partitions**, no indexes needed, backups handled via Time Travel/Fail-safe. Near-zero administration.

## 4. Modern Data Formats (JSON etc.)
- **Structured** = rows/columns (SQL tables). **Semi-structured** = flexible, labeled data (JSON, Parquet). **Unstructured** = images, video, etc.
- Traditional warehouses need a fixed **schema** before loading, so JSON had to be manually **flattened** first — broke whenever JSON shape changed.

**Snowflake:** **VARIANT** column type stores JSON/Parquet directly, no flattening needed, queried with plain SQL.

## 5. Data Silos & Hard Sharing
- **Data silo** = data locked in one team's system, hard to combine across teams.
- Sharing meant exporting CSVs via email/SFTP -> copies go **stale**, extra storage, security risk, no control once sent.

**Snowflake:** centralizes data across teams; **secure data sharing** lets a **provider** grant a **consumer** live query access (no copying), revocable anytime. Reader accounts exist for consumers without Snowflake.

## Recap Table
| Problem | Snowflake's Fix |
|---|---|
| Capacity guessing, upfront cost | Elastic cloud compute, pay-per-use |
| Storage+compute coupling → contention | Independent scaling, per-team warehouses |
| Manual maintenance (index/partition/tune) | Automated micro-partitioning, near-zero admin |
| Rigid schema, hard to load JSON | VARIANT type, no flattening |
| Data silos, risky file sharing | Centralized platform + live secure sharing |

## Quick Glossary
Capacity planning · Concurrency · Resource contention · Virtual warehouse · DBA · Index · Partitioning · Micro-partition · Schema · VARIANT · Flattening · Data silo · Provider/Consumer · Reader account

---
**Next Topic:** Cloud data warehouse vs traditional database
