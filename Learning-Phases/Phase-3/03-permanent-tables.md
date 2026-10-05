# Phase 3 | Topic 3: Permanent Tables

## What "Permanent" Specifically Guarantees
The **default and most feature-complete** of the three Standard sub-types.

1. **Full Time Travel support** — 1 day retention on Standard edition, up to 90 days on Enterprise+.
2. **Full Fail-safe protection** — an additional **7-day** recovery period after Time Travel expires. **Not self-service** — no `UNDROP`, no querying historical data during Fail-safe. Only Snowflake support can attempt recovery, typically only for genuine disaster-recovery cases, not routine mistakes.
3. **Storage cost implication (the real trade-off):** Permanent tables maintain both Time Travel AND Fail-safe copies, giving them the **highest storage cost** of the three Standard sub-types. Every update/delete creates new micro-partitions, and the old ones stick around for Time Travel + Fail-safe combined — potentially many days of historical versions, all counted toward storage billing.

## When to Use Permanent (Practical Decision)
The right choice for genuinely important data — production business data, especially anything not easily reloadable/recreatable if lost. Final, trusted tables in an `ANALYTICS` schema are the classic use case.

**Interview angle:** *"If a Permanent table's Time Travel period expires and someone realizes they deleted important data, is it unrecoverable?"*
Not immediately — it enters Fail-safe for up to 7 more days, but recovery requires contacting Snowflake support directly; not self-service via `UNDROP`.

## Hands-On: Checking Actual Retention Settings
```sql
SHOW TABLES LIKE 'ORDERS_RAW' IN SCHEMA SALES_DB.RAW;
```
Look for the **`retention_time`** column — shows the Time Travel retention period (in days) configured for that specific table. (Fail-safe isn't configurable — fixed at 7 days for Permanent tables, always.)

**Result confirmed:** `retention_time: 1` for `ORDERS_RAW` — the Standard edition default. If dropped or deleted today: 1 day of self-service Time Travel recovery, followed by 7 more days of Fail-safe (support-only), before data becomes genuinely unrecoverable.

---
**Next Topic:** Transient tables
