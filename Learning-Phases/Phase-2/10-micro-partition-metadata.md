# Phase 2 | Topic 10: Micro-Partition Metadata

## What Metadata Is Stored Per Micro-Partition
For **every single micro-partition**, Snowflake's Cloud Services Layer automatically stores metadata — a detailed label on each data "box," created with zero user effort:

- **Range of values for each column** in that partition — e.g., "this partition's `order_date` column ranges from 2026-09-01 to 2026-09-15."
- **Number of distinct values** per column (helps query planning).
- **Number of rows** in that partition.
- The actual **size** of the partition.

## Why This Metadata Matters
This metadata is the literal mechanism behind **partition pruning** (next topic): when a query runs `WHERE order_date = '2026-09-10'`, the Cloud Services Layer checks this metadata **first**, instantly identifying which micro-partitions could possibly contain matching rows, and skips the rest entirely — without ever touching their actual data.

## Reading Metadata via `SYSTEM$CLUSTERING_INFORMATION` — Real Example
Using the 5-million-row `BIG_ORDERS` table (35 micro-partitions, created in Topic 9):
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('SALES_DB.RAW.BIG_ORDERS', '(ORDER_DATE)');
```
Result:
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

**Interpreting each field:**
- **`total_partition_count: 35`** — total micro-partitions making up the table.
- **`average_overlaps: 34.0`** (high) — measures how much the value ranges of `order_date` across different partitions overlap each other. Here, data was generated with `MOD(SEQ4(), 365)`, scattering dates somewhat randomly rather than loading in date order — so rows sharing the same `order_date` ended up spread across almost all 35 partitions instead of being grouped together. A realistic outcome for data loaded without any ordering.
- **`average_depth: 35.0`** — on average, how many partitions would need to be scanned to find all rows for a given `order_date` value. A depth of 35 (out of 35 total) means a query filtering on `order_date` would likely need to scan **almost every partition** — pruning would barely help, since the dates aren't physically grouped.
- **`total_constant_partition_count`** — count of partitions where a column's value never changes (fully constant) — useful for very low-cardinality columns; 0 here since none of this table's columns are constant across a whole partition.

## Why This Matters Going Forward
High overlap + high depth = a poorly-clustered table, where filtering on that column won't benefit much from pruning. This is exactly the scenario **clustering keys** (a few topics ahead) are designed to fix, by forcing better physical organization of data within partitions — and exactly why **partition pruning** (next topic) depends entirely on this metadata being favorable.

**Interview angle:** *"How does Snowflake decide which micro-partitions to skip during a filtered query?"*
It checks the per-partition metadata (min/max value ranges per column) maintained by the Cloud Services Layer — if a partition's recorded range cannot possibly contain the filtered value, it's skipped without reading any actual data.

---
**Next Topic:** Partition pruning
