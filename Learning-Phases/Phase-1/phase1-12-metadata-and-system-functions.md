# Phase 1 | Topic 12: Metadata and System Functions

## What Metadata Means
**Metadata** = data about the Snowflake environment itself (what objects exist, who's connected, compute usage) — as opposed to actual business data (orders, customers) stored in tables.

**Two main channels for metadata:**
1. **System functions** — built-in functions called directly in a query, returning info about the current session/environment.
2. **Metadata views** (`INFORMATION_SCHEMA` and `SNOWFLAKE.ACCOUNT_USAGE`) — queryable tables for broader/historical metadata.

## Context Functions (current session info)
```sql
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), 
       CURRENT_SCHEMA(), CURRENT_USER(), CURRENT_TIMESTAMP();
```
**Interview angle:** *"How to confirm which role a stored procedure runs under?"* Call `CURRENT_ROLE()` inside its logic — useful for debugging permission issues.

## Object, Query, and Usage System Functions
**Object-related:**
```sql
SELECT SYSTEM$CLUSTERING_INFORMATION('SALES_DB.ANALYTICS.DAILY_SALES');
```
Checks how well a table's data is organized for performance (used more in the clustering/performance topic).

**Query-related:**
```sql
SELECT LAST_QUERY_ID();
SELECT SYSTEM$CANCEL_QUERY('<query_id>');
```
`LAST_QUERY_ID()` = ID of the most recent query run. `SYSTEM$CANCEL_QUERY` = stops a specific query by ID — key for killing an accidental, expensive runaway query.

**Performance-related:**
```sql
SELECT SYSTEM$ESTIMATE_QUERY_ACCELERATION('<query_id>');
```
Checks if Query Acceleration (a warehouse borrowing extra temporary compute for one heavy query) would help a slow query.

**Interview angle:** *"A teammate's query has run 20 minutes, burning credits. What do you do?"* Find its query ID (Query History in Snowsight, or `LAST_QUERY_ID()`), then run `SYSTEM$CANCEL_QUERY('<query_id>')`.

## The Two Metadata View Systems

| | `INFORMATION_SCHEMA` | `SNOWFLAKE.ACCOUNT_USAGE` |
|---|---|---|
| Scope | Per-database | Entire account |
| Data | Live, current-moment only | Historical (up to ~1 year depending on view) |
| Deleted objects | Not shown | Still shown |
| Typical use | "What does my database look like right now" | Cost monitoring, auditing, security review over time |

```sql
-- INFORMATION_SCHEMA example
SELECT * FROM SALES_DB.INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'ORDERS_RAW';

-- ACCOUNT_USAGE examples
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY;
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY;  -- credit consumption per warehouse over time
```

**Interview angle:** *"Which warehouse consumed the most credits last month?"* `SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` — `INFORMATION_SCHEMA` can't answer this since it has no historical usage trends.

---
**Phase 1 complete.**
