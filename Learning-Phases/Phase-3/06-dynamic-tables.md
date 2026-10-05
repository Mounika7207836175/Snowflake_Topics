# Phase 3 | Topic 6: Dynamic Tables

## The Problem This Solves
Traditionally, keeping a transformed/aggregated table up to date required manually: writing a **Task** (scheduled job) that runs on a timer, writing the SQL logic to recompute/merge new results, and managing incremental-processing logic yourself (tracking what's new since the last run). Real, manual pipeline-building work.

## What a Dynamic Table Does Instead
Write **one query** describing the desired result, tell Snowflake how often to refresh it, and Snowflake **automatically** handles scheduling and incremental recomputation behind the scenes.

```sql
CREATE DYNAMIC TABLE SALES_DB.ANALYTICS.CUSTOMER_TOTALS
  TARGET_LAG = '1 minute'
  WAREHOUSE = COMPUTE_WH
AS
SELECT customer_id, SUM(amount) AS total_spent
FROM SALES_DB.RAW.ORDERS_RAW
GROUP BY customer_id;
```

**Syntax breakdown:**
- **`TARGET_LAG = '1 minute'`** — a goal, not a literal fixed schedule: "I'm okay with this data being up to 1 minute behind the source." Snowflake decides on its own how frequently to actually refresh to hit this target.
- **`WAREHOUSE = COMPUTE_WH`** — which warehouse runs the refresh computations.
- The `AS SELECT ...` defines the table's content — similar to a View, but the result is **physically stored and refreshed on a schedule**, not recomputed live on every query.

## Dynamic Table vs View vs Standard Table

| | View | Dynamic Table | Standard Table |
|---|---|---|---|
| When computed | Live, every single query | On its own background refresh schedule (per `TARGET_LAG`) | Whenever the user's own INSERT/UPDATE/MERGE/Task runs it |
| What a query reads | Freshly computed result, always | Already-computed, stored result from the last refresh | Stored data as of last write |
| Speed | Can be slow for complex/heavy queries | Fast — just a table read | Fast — just a table read |
| Freshness | Always fresh | Can lag by up to the target lag | As fresh as the last write |

**The critical mechanism, precisely:** querying a Dynamic table **never triggers computation**. The aggregation query only runs in the **background**, on Snowflake's own refresh schedule, completely separate from when anyone happens to query it. A user's `SELECT` just reads whatever the most recent automatic refresh already produced and stored.

**Interview angle:** *"Why use a Dynamic table instead of a View?"*
A View recomputes its full query live on every access — slow for complex aggregations over large data. A Dynamic table pre-computes and stores the result on a schedule, so queries are fast — trading a small amount of freshness (the target lag) for significantly better query performance.

**Interview angle:** *"Does querying a Dynamic table trigger its underlying query to run?"*
No — the underlying query only runs on Snowflake's own background refresh schedule. Querying the table just reads the already-computed, stored result from the last refresh.

## Hands-On: Watching a Dynamic Table Refresh Automatically

```sql
-- Create it
CREATE DYNAMIC TABLE SALES_DB.ANALYTICS.CUSTOMER_TOTALS
  TARGET_LAG = '1 minute'
  WAREHOUSE = COMPUTE_WH
AS
SELECT customer_id, SUM(amount) AS total_spent
FROM SALES_DB.RAW.ORDERS_RAW
GROUP BY customer_id;

-- Query it
SELECT * FROM SALES_DB.ANALYTICS.CUSTOMER_TOTALS;

-- Insert new data into the SOURCE table (not the dynamic table)
INSERT INTO SALES_DB.RAW.ORDERS_RAW (order_id, customer_id, order_date, amount)
VALUES ('5001', 'C001', '2026-10-05', '500.00');

-- Wait 60-90 seconds (to exceed target lag), then query again
SELECT * FROM SALES_DB.ANALYTICS.CUSTOMER_TOTALS;

-- Check refresh history (note: needs full database path, like any INFORMATION_SCHEMA call)
SELECT * FROM TABLE(SALES_DB.INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(
  NAME => 'SALES_DB.ANALYTICS.CUSTOMER_TOTALS'
));
```

**Results confirmed:**
- Refresh history showed **19 separate automatic refreshes**, all `STATE: SUCCEEDED` — all triggered by Snowflake's own background process, never manually.
- After inserting two $500 orders for `C001`, `total_spent` increased by **1000 total** — automatically picked up by the background refresh, with no direct interaction with `CUSTOMER_TOTALS` itself.

**Debugging note:** `DYNAMIC_TABLE_REFRESH_HISTORY` is an `INFORMATION_SCHEMA` function and needs the database prefix specified (`SALES_DB.INFORMATION_SCHEMA...`), same as every other `INFORMATION_SCHEMA` call.

---
**Next Topic:** External tables
