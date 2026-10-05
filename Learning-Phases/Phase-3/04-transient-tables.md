# Phase 3 | Topic 4: Transient Tables

## What Makes a Table "Transient"
The second Standard sub-type — for data important enough to keep, but not critical enough to need full Permanent-table protection (and cost).

**Exact trade-offs vs Permanent:**
1. **Time Travel:** maximum of **1 day**, regardless of Snowflake edition — even on Enterprise, Transient never gets the extended 90-day option. A hard ceiling, not a default that can be raised.
2. **Fail-safe:** **zero** — completely absent. Once Time Travel expires (max 1 day), the data is gone, with no support-recoverable safety net.
3. **Storage cost:** significantly cheaper than Permanent, since there's no Fail-safe copy maintained and Time Travel is capped short.

## What "Staging" Means (New Concept)
In real data pipelines, data typically moves through stages before being ready for business use:
1. **Raw landing** — data arrives exactly as the source sent it, untouched, often messy.
2. **Staging** — data is cleaned, reformatted, prepared, but still a work-in-progress copy, not the final trusted version.
3. **Final/Analytics** — the clean, trusted data that reports and dashboards actually use.

**Why staging data fits Transient tables well:** staging data is typically **reloaded fresh every pipeline run**. If a run has a bug, the fix is to re-run the pipeline from the raw source again — not to "restore" yesterday's staging data. Since staging data is disposable and easily regenerated, there's no need to pay for long Time Travel or Fail-safe protection on it.

**Interview angle:** *"Why would an engineer deliberately choose a Transient table over Permanent, despite less protection?"*
For intermediate/reloadable data (staging tables, temporary processing results) where cost savings matter more than long recovery options — if data can be easily regenerated from its source, extra protection is unnecessary expense.

**Interview angle:** *"Why might staging tables in a pipeline typically be created as Transient rather than Permanent?"*
Staging data is usually disposable and regenerated on each run, so reduced protection (1-day Time Travel, no Fail-safe) is an acceptable trade-off for lower storage cost.

## Hands-On: Creating a Real Transient Table

```sql
-- Step 1: create
CREATE TRANSIENT TABLE SALES_DB.RAW.ORDERS_STAGING_TEST (
    order_id STRING,
    customer_id STRING,
    amount STRING
);

-- Step 2: insert a test row
INSERT INTO SALES_DB.RAW.ORDERS_STAGING_TEST VALUES ('T001', 'C001', '100.00');

-- Step 3: check retention settings
SHOW TABLES LIKE 'ORDERS_STAGING_TEST' IN SCHEMA SALES_DB.RAW;

-- Step 4: confirm type via GET_DDL
SELECT GET_DDL('TABLE', 'SALES_DB.RAW.ORDERS_STAGING_TEST');
```

**Results confirmed:**
- `retention_time: 1` (the max allowed for Transient).
- `GET_DDL` output explicitly showed `CREATE OR REPLACE TRANSIENT TABLE` — unlike a Permanent table's DDL, which has no prefix keyword at all.

---
**Next Topic:** Temporary tables
