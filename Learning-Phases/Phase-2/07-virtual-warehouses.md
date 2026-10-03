# Phase 2 | Topic 7: Virtual Warehouses (Full Depth + Hands-On)

## What a Virtual Warehouse Is
A **Virtual Warehouse** is Snowflake's unit of **compute power** — the engine that does work for any query, data load, or operation touching data.

**"Virtual"** because it's not a physical machine you see or manage — it's a logical, on-demand allocation of CPU, memory, and temporary disk that Snowflake provisions from its underlying cloud infrastructure.

**Every data operation needs an active warehouse:** `SELECT`, `COPY INTO`, `INSERT`/`UPDATE`/`DELETE`, creating tables/views — none of these can run without one (even if it auto-resumes in a fraction of a second).

## Warehouse States
1. **Started/Running** — actively available, executes queries immediately.
2. **Suspended** — not running, **zero credits** consumed, but configuration (size, settings) is remembered.
3. **Resuming** — brief transition when a suspended warehouse starts back up (manual, or automatic via `AUTO_RESUME = TRUE` when a new query arrives).

**Auto-suspend / auto-resume mechanics:**
- `AUTO_SUSPEND = 60` → suspends automatically after 60 seconds with no running query; billing stops immediately.
- `AUTO_RESUME = TRUE` → resumes automatically the moment a new query arrives (typically 1-2 seconds for smaller sizes).
- **Default pre-created warehouses** (like `COMPUTE_WH` on a trial account) may have a longer default auto-suspend (commonly 600 seconds/10 min) unless explicitly changed — always verify with `SHOW WAREHOUSES` rather than assuming.

## Hands-On: Checking and Observing Warehouse States
```sql
SHOW WAREHOUSES;                          -- lists all warehouses, their size, state, auto-suspend/resume settings
SHOW WAREHOUSES LIKE 'marketing_wh';      -- check one warehouse's exact settings
```

**Live resume → suspend cycle test:**
```sql
USE WAREHOUSE marketing_wh;
SELECT * FROM SALES_DB.RAW.ORDERS_RAW;    -- wakes the warehouse (STARTED)
-- wait 70-90 seconds (longer than AUTO_SUSPEND setting)
SHOW WAREHOUSES LIKE 'marketing_wh';      -- should now show SUSPENDED
```

## Two Different Caches — A Key Distinction

| | Local Disk Cache (Warehouse Cache) | Result Cache |
|---|---|---|
| **Lives in** | Inside the warehouse — on its servers' local SSD disks | Cloud Services Layer — not inside any warehouse |
| **Requires active warehouse?** | Yes — wiped when the warehouse suspends | No — works even with zero active warehouses |
| **What it stores** | Recently-scanned micro-partitions (raw data) | The final computed result set of a specific query |
| **Retention** | Lost on suspend/resize | 24 hours |
| **Triggers on** | Re-scanning the same partitions for a query that still needs processing | Running the **exact same query** again (identical SQL text, no underlying data change) |

**Why re-running an identical query didn't wake the warehouse:** the Result Cache, maintained by the Cloud Services Layer, recognized the identical query and returned the stored final answer directly — no warehouse involvement needed at all, since there was nothing to scan or compute.

**Why changing the query (adding a WHERE filter) did wake the warehouse:** it was a different query, not an exact match in the Result Cache, so Snowflake had to actually resume the warehouse and run a real TableScan → Filter → Result plan to compute a fresh answer.

**Reading a Query Profile for this:** a Result-Cache hit typically shows near-zero duration and no real execution plan steps. A real execution (like the filtered query) shows a full plan with `Total execution time`, `Partitions scanned`, and a `Percentage scanned from cache` stat — note this percentage refers to the **local disk cache (Cache #1)**, not the Result Cache.

**Interview angle:** *"Can Snowflake return a query result without starting any warehouse? How?"*
Yes — via the Result Cache, maintained by the Cloud Services Layer. If an identical query ran within the last 24 hours and underlying data hasn't changed, Snowflake returns the cached result directly, with zero compute involvement.

## Hands-On: Resizing a Warehouse
```sql
SHOW WAREHOUSES LIKE 'marketing_wh';                      -- check current size
ALTER WAREHOUSE marketing_wh SET WAREHOUSE_SIZE = 'SMALL'; -- resize up
SHOW WAREHOUSES LIKE 'marketing_wh';                      -- confirm change
ALTER WAREHOUSE marketing_wh SET WAREHOUSE_SIZE = 'XSMALL';-- resize back down (save credits)
```

**What happens during a resize:** Snowflake provisions additional servers (e.g., 1 server at X-Small → 2 at Small) and adds them to the warehouse. A query **actively running** during a resize generally **continues on its original server(s)** — the new size takes effect starting from the **next** query, not retroactively. No downtime, unlike traditional database resizing.

**Interview angle:** *"If you resize a warehouse from Small to Large while a query is running, what happens to that query?"*
It continues on its original compute; the larger size applies from the next submitted query — no interruption or downtime.

---
**Next Topic:** Columnar storage and compression
