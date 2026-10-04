# Phase 2 | Topic 11: Partition Pruning

## What Pruning Means
**Partition pruning** = the Cloud Services Layer **skips reading micro-partitions entirely** when it can determine, from metadata alone, that a partition **cannot possibly** contain rows matching a query's filter — without ever opening or scanning that partition's actual data.

## The Exact Mechanism
1. A query runs with a `WHERE` clause (e.g., `WHERE order_date = '2026-09-10'`).
2. Cloud Services checks the **min/max value range metadata** (Topic 10) for that column, on every micro-partition.
3. For each partition: *"Could this partition possibly contain a matching row, given its recorded min/max range?"*
4. If **no** (range doesn't overlap the filter value), that partition is **completely skipped** — zero bytes read.
5. Only partitions that **could** contain a match get physically scanned.

**Why it's powerful:** on a table with thousands of partitions, a well-targeted filter might scan just 2-3 instead of all of them — massive reduction in data read, faster queries, lower compute cost.

**Critical dependency:** pruning only works well when data is physically organized so similar values cluster together in the same few partitions. A high `average_depth` (Topic 10) for a column means poor pruning on that column — queries will scan most/all partitions regardless of the filter.

**Interview angle:** *"If a table has a high `average_depth` for a column, what does that mean for query performance when filtering on it?"*
Poor pruning — queries filtering on that column will scan most or all partitions, since values aren't physically grouped, limiting pruning's benefit.

## Hands-On: Good vs Bad Pruning on the Same Table (`BIG_ORDERS`)

**Test 1 — Filter on `order_date` (poor pruning, as predicted by Topic 10's metadata):**
```sql
SELECT * FROM SALES_DB.RAW.BIG_ORDERS WHERE order_date = '2025-06-15';
```
**Result:** 35 of 35 partitions scanned — essentially no pruning.

**Test 2 — Filter on `order_id` (excellent pruning):**
```sql
SELECT * FROM SALES_DB.RAW.BIG_ORDERS WHERE order_id = 2500000;
```
**Result:** 1 of 35 partitions scanned.

## Why the Two Columns Behaved So Differently

**`order_date` — poor pruning, root cause:** generated via `MOD(SEQ4(), 365)`, which jumps around (row 1 might get day 45, row 2 day 310, row 3 day 12...). As rows were written into partitions **in the order generated**, wildly different dates ended up sitting in the same partition — e.g., partition 1 might hold dates from January, June, and December all mixed together. This is why every partition's date range overlaps heavily with every other's.

**`order_id` — excellent pruning, root cause: Natural Clustering.** No explicit clustering command was ever run. `SEQ4()` generates strictly increasing values (1, 2, 3, 4...), and Snowflake writes rows into micro-partitions **in the order they arrive** during insert. So partition 1 naturally held roughly `order_id` 1-142,857, partition 2 roughly 142,858-285,714, and so on — purely a side effect of insertion order, not a deliberate clustering action.

**The core lesson:** both columns live on the same table, but one pruned perfectly and one didn't — entirely due to **the order data happened to be loaded in** for each column, not due to any explicit setup. Clustering keys (a later topic) become necessary only when natural insertion order doesn't keep the column you need to filter on well-organized.

**Interview angle:** *"If a column has good pruning without any explicit clustering key, what's actually happening?"*
Natural clustering — data happened to be physically grouped by that column simply due to insertion order, with no manual intervention.

---
**Next Topic:** Natural data clustering
