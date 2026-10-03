# Phase 2 | Topic 2: Storage Layer

## What Happens to Data Physically — Step by Step
1. Incoming data is **compressed** using Snowflake's internal columnar compression algorithms.
2. Data is reorganized into a **columnar format** (stored column-by-column, not row-by-row like traditional databases). *(Deep dive in the "Columnar storage" topic.)*
3. The compressed, columnar data is split into small chunks called **micro-partitions** (typically 50-500 MB of uncompressed data per partition).
4. Micro-partitions are physically stored in the **cloud provider's own blob/object storage** underneath Snowflake (Azure Blob Storage, S3, or GCS depending on account) — fully managed, with zero user control over file placement, compression, or partition organization.

**Interview angle:** *"If I run `CREATE TABLE`, where does the data physically end up?"*
In the cloud provider's blob storage, organized into Snowflake-managed micro-partitions — not any system the user directly accesses.

## Inspecting Micro-Partitions (Hands-On)
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('SALES_DB.RAW.ORDERS_RAW', '(ORDER_DATE)');
```
**Important:** this function requires either an explicit clustering key already set on the table, or a column passed as a second argument to check natural clustering against. Running it with just the table name fails with *"Invalid clustering keys or table is not clustered"* if no clustering key exists yet.

**Result on a small practice table:**
- `total_partition_count: 1` — entire table fits in a single micro-partition (expected for tiny data; partitions only split once data grows into the 50-500 MB range).
- `average_overlaps: 0` — measures how much value ranges across partitions overlap; with only one partition, there's nothing to overlap with, so it's trivially 0.

**Why this matters (for later):** this function becomes meaningful on large tables with many micro-partitions. A **low** `average_overlaps` means Snowflake can prune (skip) most partitions efficiently during a filtered query. A **high** value means heavy overlap, so Snowflake can't skip much and scans more than necessary. Revisited properly in the "Partition pruning" and "Clustering keys" topics.

**Table storage stats:**
```sql
SELECT TABLE_NAME, ROW_COUNT, BYTES 
FROM SALES_DB.INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME = 'ORDERS_RAW';
```
Shows actual compressed storage size (`BYTES`) Snowflake is using for the table.

---
**Next Topic:** Compute layer
