# Phase 2 | Topic 16: Metadata Cache

## What It Stores
The third of Snowflake's three caches, storing **statistics and metadata about tables and micro-partitions** — distinct from Result Cache (final query answers) and Warehouse Cache (raw micro-partition data).

**Precisely what lives here:**
- Table-level stats: row counts, table sizes.
- Micro-partition-level stats: min/max value ranges per column (Topic 10) — the exact data powering partition pruning.
- Column statistics: distinct value counts, null counts.

**Where it lives:** the **Cloud Services Layer**, same as Result Cache — NOT inside any warehouse, so it requires **zero active warehouse** to access.

## Why It Exists
Without this cache, every query would need to physically scan micro-partition headers/metadata from storage just to decide which partitions to prune — adding real latency before actual scanning begins. Keeping this metadata cached lets Snowflake make pruning decisions almost instantly, without touching storage for that decision-making step.

## The Key Connection
This is exactly why `INFORMATION_SCHEMA` queries (and `SYSTEM$CLUSTERING_INFORMATION`) work with **no active warehouse** — they read directly from this Metadata Cache in the Cloud Services Layer, not from any warehouse.

**Interview angle:** *"Why can you run `SELECT * FROM INFORMATION_SCHEMA.TABLES` without an active warehouse, but not `SELECT * FROM my_actual_table`?"*
`INFORMATION_SCHEMA` reads from the Metadata Cache (Cloud Services Layer, no warehouse needed). Querying actual table data requires scanning real micro-partitions, which can only happen via an active virtual warehouse.

## Hands-On: Proving Metadata Queries Need No Warehouse

```sql
-- Step 1: suspend all warehouses
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
ALTER WAREHOUSE marketing_wh SUSPEND;

-- Step 2: metadata-only query
SELECT TABLE_NAME, ROW_COUNT, BYTES 
FROM SALES_DB.INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME = 'BIG_ORDERS';

-- Step 3: check warehouse states
SHOW WAREHOUSES;

-- Step 4: contrast test — actual data query
SELECT * FROM SALES_DB.RAW.BIG_ORDERS LIMIT 5;
```

**Result confirmed:** Step 2 returned results instantly while Step 3 showed all warehouses still `SUSPENDED` — proving the metadata query never needed to wake anything up. Step 4 (real data scan) forced a warehouse to resume, since it needs to read actual micro-partitions, not just metadata.

## Three Caches — Final Complete Comparison

| | Local Disk Cache (Warehouse Cache) | Result Cache | Metadata Cache |
|---|---|---|---|
| **Lives in** | Inside a warehouse (local SSD) | Cloud Services Layer | Cloud Services Layer |
| **Stores** | Raw micro-partition data | Final computed query result | Table/partition/column statistics |
| **Needs active warehouse?** | Yes | No | No |
| **Lost/invalidated when** | Warehouse suspends/resizes | After 24 hours, or if data changes | Updates automatically as data changes |

---
**Next Topic:** How Snowflake processes a query (Phase 2 capstone topic)
