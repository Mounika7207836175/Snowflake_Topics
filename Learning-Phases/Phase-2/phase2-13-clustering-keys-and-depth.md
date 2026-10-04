# Phase 2 | Topic 13: Clustering Keys and Clustering Depth

## What a Clustering Key Does
A **clustering key** is an explicit instruction telling Snowflake: "keep this table's data physically organized by this column, even as updates/inserts happen over time." Unlike natural clustering (a byproduct of insertion order), this is a **deliberate, ongoing commitment** Snowflake maintains.

```sql
ALTER TABLE SALES_DB.RAW.BIG_ORDERS CLUSTER BY (order_id);
```

**What happens behind the scenes:**
1. Snowflake does **not** instantly reorganize the entire table when the command runs.
2. It starts a **background service** that gradually reorganizes existing micro-partitions to better align with the clustering key, over time.
3. Going forward, this service continuously monitors the table and **automatically re-clusters** it whenever new inserts/updates degrade the organization.
4. This is **Automatic Reclustering** — runs on Snowflake's own **serverless compute**, NOT the user's virtual warehouse credits. Billed separately, as its own line item, based on the compute Snowflake spends reorganizing the table.

**"Over time" — how long reclustering actually takes:** no fixed timer. The background service continuously monitors degradation (e.g., `average_depth`); when degradation crosses an internal threshold Snowflake considers worth fixing, it schedules reclustering work. Duration depends on table size and how much reorganization is needed — small/minor degradation might resolve in minutes to hours; large/heavy degradation may take longer and happen in batches. Snowflake is deliberately conservative: very minor degradation (like a handful of rows out of millions) may not get prioritized at all, since the reclustering cost could outweigh the tiny benefit.

## Clustering Depth — Formal Definition
**Clustering depth for a given row** = the number of micro-partitions that overlap with the partition containing that row, for the column being checked. `average_depth` (seen throughout Topics 9-12) is this number averaged across the whole table. **1.0 = theoretical best** (zero overlapping partitions); higher = progressively worse organization.

**Interview angle:** *"Does Automatic Reclustering use your virtual warehouse's compute?"*
No — runs on Snowflake-managed serverless compute, billed separately from warehouse credits.

## Does Reclustering Break Time Travel? (No — and why)
Reclustering follows the **exact same immutability rule** as any other write:
- It does **not** destroy old micro-partitions directly. Like an `UPDATE`, it creates **new, better-organized** micro-partitions and marks the **old** ones inactive for current queries.
- Old, inactive micro-partitions **still physically exist** and remain available for the **Time Travel retention period** (1 day Standard, up to 90 Enterprise) — same mechanism as Topic 9.
- Querying `BIG_ORDERS AT (OFFSET => -3600)` can still reconstruct the historical state, using whichever micro-partitions (pre- or post-reclustering) were active at that point.

**Takeaway:** reclustering is just another write operation from storage's perspective — never deletes anything immediately. Same "create new, deactivate old" pattern as updates/deletes.

**Interview angle:** *"Does automatic reclustering affect Time Travel on a table?"*
No — it creates new micro-partitions and deactivates old ones, following the same immutable write pattern as any update. Old partitions remain available for the Time Travel window.

## Natural Clustering vs Clustering Keys — The Real Difference

**What's IDENTICAL between them:** both follow the exact same immutability rule — any write (a normal `UPDATE`, or reclustering) creates new micro-partitions and marks old ones inactive, retained for Time Travel. This is universal to all Snowflake writes, regardless of whether a clustering key exists.

**Where the actual difference lies — WHO initiates reorganization, and HOW OFTEN:**
- **Without a clustering key (natural clustering only):** reorganization happens **only** as a side effect of the user's own operations (`UPDATE`, `DELETE`, loading new data). Snowflake never proactively goes back to fix degraded natural clustering — degradation accumulates indefinitely unless the user manually intervenes (e.g., rewriting the table).
- **With a clustering key set:** Snowflake's background service **actively and proactively** creates new, better-organized micro-partitions **on its own initiative** — even with no further user `UPDATE`/`INSERT` — specifically to undo degradation and keep `average_depth` low, because the clustering key was requested.

**Clean summary:** natural clustering reorganizes data only as an incidental side-effect of the user's own writes. A clustering key causes Snowflake to actively and continuously do *extra* reorganization work, beyond what the user's own writes would cause, specifically to maintain good organization — which is exactly what gets billed as Automatic Reclustering credits.

**Interview angle:** *"If natural clustering and clustering keys both create new partitions and deactivate old ones the same way, what's the actual difference?"*
The mechanism is identical either way. The difference is who initiates the work: natural clustering only reorganizes as a side effect of the user's own writes; a clustering key makes Snowflake proactively and continuously do additional reorganization work on its own — hence the separate ongoing cost.

## Hands-On (to run and verify next session)
```sql
-- Step 1: Apply clustering key
ALTER TABLE SALES_DB.RAW.BIG_ORDERS CLUSTER BY (order_id);

-- Step 2: Check clustering state
SELECT SYSTEM$CLUSTERING_INFORMATION('SALES_DB.RAW.BIG_ORDERS', '(ORDER_ID)');

-- Step 3: Check actual reclustering activity/history
SELECT * FROM TABLE(INFORMATION_SCHEMA.AUTOMATIC_CLUSTERING_HISTORY(
  DATE_RANGE_START => DATEADD('day', -1, CURRENT_DATE()),
  TABLE_NAME => 'BIG_ORDERS'
));
```
**Expectation to set:** since reclustering is gradual and Snowflake is conservative about minor degradation, immediate improvement may not be visible right after running `ALTER TABLE` on this small test table — an empty result from the clustering history function is a normal, expected outcome here, not a failure.

---
**Next Topic:** Result cache
