# Phase 2 | Topic 14: Result Cache

## Precise Definition
The Result Cache stores the **final computed output** of a query — not raw data, not micro-partitions, but the literal rows that would be returned — at the **Cloud Services Layer** level, independent of any specific warehouse.

## Exact Conditions for a Cache Hit (all must be true)
1. **SQL text must match exactly** (whitespace is usually normalized; identifier case sensitivity can matter depending on how objects were created — see Topic 9's case sensitivity notes).
2. **Underlying table's data must not have changed** since the cached result was stored — any insert/update/delete on the queried table invalidates the cache for that query.
3. Query must have been run within the last **24 hours**.
4. **Role running the query must have the same access rights** — a different role that would see different data (e.g., due to row-level security) won't reuse another role's cached result.

## What Happens on a Cache Hit
The Cloud Services Layer directly returns the stored result. **No warehouse resumes, no micro-partitions are read, no compute credits are consumed.** This is exactly why a suspended warehouse can stay suspended even after an identical query "runs" again (discovered hands-on in the Virtual Warehouses topic).

**Interview angle:** *"If you run the exact same query twice in a row, will the second show a non-zero duration in Query History?"*
It still appears in Query History (every query is logged), but execution time is near-instant (a few ms), and its Query Profile indicates a cache-sourced result rather than a real TableScan/Filter/Result plan.

## Hands-On: Testing Cache Invalidation (to run)
```sql
-- Step 1
SELECT COUNT(*) FROM SALES_DB.RAW.ORDERS_RAW;

-- Step 2: identical query again — expect cache hit, near-instant duration
SELECT COUNT(*) FROM SALES_DB.RAW.ORDERS_RAW;

-- Step 3: change the underlying data
INSERT INTO SALES_DB.RAW.ORDERS_RAW (order_id, customer_id, order_date, amount)
VALUES ('9999', 'C999', '2026-10-01', '50.00');

-- Step 4: identical query again — expect FRESH execution now, cache invalidated
SELECT COUNT(*) FROM SALES_DB.RAW.ORDERS_RAW;
```
Compare Query History durations: Step 2 should be near-instant (cache hit); Step 4 should show real execution (cache invalidated by the insert).

## Controlling the Result Cache Manually
Useful for honest performance benchmarking — a cached result would otherwise hide true execution time.

```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;   -- force fresh execution for all queries this session
ALTER SESSION SET USE_CACHED_RESULT = TRUE;    -- turn caching back on
```

**Interview angle:** *"You're benchmarking a query after adding a clustering key, but results seem suspiciously fast. What might be wrong?"*
The Result Cache may be returning a stored result from before the change. Set `ALTER SESSION SET USE_CACHED_RESULT = FALSE` first to get an honest, fresh execution time.

## Three Caches — Quick Comparison

| | Local Disk Cache (Warehouse Cache) | Result Cache |
|---|---|---|
| **Lives in** | Inside a warehouse (local SSD) | Cloud Services Layer |
| **Stores** | Raw micro-partition data | Final computed query result |
| **Needs active warehouse?** | Yes | No |
| **Lost when?** | Warehouse suspends/resizes | After 24 hours, or if data changes |

(Local Disk Cache = Warehouse Cache, same thing, interchangeable names. Metadata Cache is the third type — covered next.)

---
**Next Topic:** Warehouse cache
