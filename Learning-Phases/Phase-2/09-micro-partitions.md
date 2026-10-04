# Phase 2 | Topic 9: Micro-Partitions (Full Depth + Hands-On)

## How Data Gets Split Into Micro-Partitions
When data is loaded (`COPY INTO`, `INSERT`, etc.), Snowflake automatically:
1. Takes incoming rows.
2. Groups them into batches of roughly **50-500 MB of uncompressed data** per batch.
3. Compresses each batch using columnar compression (see Topic 8).
4. Stores each compressed batch as one **micro-partition** — an immutable, physical unit of storage.

## Immutability — A Critical, Often-Tested Detail
Once written, a micro-partition is **never edited in place**. An `UPDATE` or `DELETE` on even one row causes Snowflake to:
1. Identify which micro-partition(s) contain the affected row(s).
2. Create a **brand new** micro-partition with the updated/remaining data.
3. Mark the **old** micro-partition inactive for current queries (not instantly deleted — it persists for the Time Travel retention period, enabling querying "data as it was before").

**Why immutability matters:** simplifies the storage layer and enables features like Time Travel and zero-copy cloning almost "for free," since old data versions aren't destroyed immediately — they're just no longer referenced by the "current" table version.

**Interview angle:** *"If you update one row, does Snowflake modify it in place?"*
No — it creates a new micro-partition with the updated data; the old one becomes inactive (but persists for Time Travel) rather than being edited directly.

## Hands-On: Creating a Real Multi-Partition Table

Small practice tables (a handful of rows, or even 5 million tiny rows ≈ 150MB) can still fit inside a **single** micro-partition, since partitions hold up to ~500MB uncompressed. To see real partitioning, row size needs to be large enough to push total data past that threshold.

```sql
CREATE OR REPLACE TABLE SALES_DB.RAW.BIG_ORDERS AS
SELECT 
    SEQ4() AS order_id,
    'C' || MOD(SEQ4(), 500) AS customer_id,
    DATEADD(DAY, MOD(SEQ4(), 365), '2025-01-01') AS order_date,
    UNIFORM(10, 5000, RANDOM()) AS amount,
    RANDSTR(300, RANDOM()) AS padding_text
FROM TABLE(GENERATOR(ROWCOUNT => 5000000));
```
`SEQ4()` generates a sequence number; `GENERATOR` is Snowflake's built-in row-generating function; `RANDSTR(300, ...)` adds a ~300-character text column to push total uncompressed size well past 500MB (~1.6GB total here), guaranteeing multiple partitions.

**Checking partition count:**
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('SALES_DB.RAW.BIG_ORDERS', '(ORDER_DATE)');
```

**Debugging tip:** if the result still shows `total_partition_count: 1` right after creating a large table, don't assume the concept is wrong — verify with actual stats before troubleshooting further:
```sql
SELECT TABLE_NAME, ROW_COUNT, BYTES FROM SALES_DB.INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'BIG_ORDERS';
SELECT COUNT(*) FROM SALES_DB.RAW.BIG_ORDERS;
DESCRIBE TABLE SALES_DB.RAW.BIG_ORDERS;
SELECT order_id, LENGTH(padding_text) AS text_length, padding_text FROM SALES_DB.RAW.BIG_ORDERS LIMIT 5;
```
A stale/cached metadata read shortly after a `CREATE OR REPLACE` can show outdated partition info — re-running the clustering check after confirming the table's real row count/content resolves this.

## Real Result Obtained
```json
{
  "cluster_by_keys" : "LINEAR(order_date)",
  "total_partition_count" : 35,
  "total_constant_partition_count" : 0,
  "average_overlaps" : 34.0,
  "average_depth" : 35.0,
  "version" : "OPTIMA"
}
```
The ~1.6GB table was automatically split into **35 micro-partitions** — purely automatic, no manual setup. (Interpreting `average_overlaps` and `average_depth` in detail is covered in Topic 10: Micro-Partition Metadata.)

---
**Next Topic:** Micro-partition metadata
