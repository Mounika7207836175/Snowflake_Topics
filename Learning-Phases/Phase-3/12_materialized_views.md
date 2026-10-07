# Snowflake: Materialized Views (MV)

## 1. What and Why

A **materialized view** is a view whose result is **physically stored** and kept up to date automatically by Snowflake.

| | Standard View | Materialized View |
|---|---|---|
| Stores | Only the query text | The query **result** (as data) |
| On each select | Re-runs query on base table | Reads the saved result |

```sql
CREATE MATERIALIZED VIEW SALES_BY_REGION_MV AS
SELECT region, SUM(amount) AS total_sales
FROM BIG_ORDERS
GROUP BY region;
```

**Key ideas**
- **Auto-maintained**: a background service refreshes it when the base table changes. No manual refresh.
- **Never stale**: if the MV is slightly behind, Snowflake combines the stored result with the newest base-table changes.
- **Costs extra**: storage (saved result) + compute credits (background refresh).
- **Edition**: requires **Enterprise edition or higher**.
- **Why use it**: makes repeated, expensive queries (aggregations, filters on huge tables) faster and cheaper to read.

---

## 2. Limitations and When NOT to Use

**Limitations**
- Query can read from **only one table**. **No joins** (including self-joins).
- Cannot be built on another view or another MV, only on a table.
- Allowed aggregates: `SUM`, `COUNT`, `MIN`, `MAX`, `AVG`. **No `HAVING`, no window functions.**
- **No non-deterministic functions** (e.g. `CURRENT_TIMESTAMP()`, `RANDOM()`).
- Restrictions on `ORDER BY`, `LIMIT`, `UNION`, and some forms of `DISTINCT`.
- **Read-only**: no `INSERT`/`UPDATE` on the MV itself.

**When NOT to use an MV**
- Base table **changes very frequently** (refresh runs constantly and burns credits).
- Query needs **joins** or complex logic (use a **dynamic table** instead).
- Base table is **small** (a normal query is already fast).
- Query is **rarely run** (you pay for refresh nobody reads).

**Good fit**: a large table that changes slowly, with the same aggregation or filter queried repeatedly.

> Note: MVs work well on **big data**. The problem is big data that **changes frequently**.

---

## 3. MV vs Secure View

| | Materialized View | Secure View |
|---|---|---|
| Purpose | Speed (performance) | Security (data protection) |
| Stores data? | Yes | No (query text only) |
| Extra cost | Storage + refresh credits | None extra |
| Hides definition? | No | Yes |
| Joins allowed? | No | Yes |

**What a secure view does**
- **Hides its definition**: with a standard view, anyone with access can see the query via `SHOW VIEWS` or `GET_DDL`. In a secure view, only the owner role can.
- **Blocks optimizer shortcuts**: disables filter pushdown that could leak data through error messages or timing. Slightly slower, but safer.
- **Required for data sharing** with another Snowflake account.

```sql
CREATE SECURE VIEW CUSTOMER_PUBLIC_V AS
SELECT customer_id, region
FROM CUSTOMERS;
```

**Remember**
- MV: "make it fast" (stores results)
- Secure view: "keep it private" (hides logic)
- You can combine them: `CREATE SECURE MATERIALIZED VIEW`.

---

## 4. Maintenance, Cost and Management

- **Refresh** is done by Snowflake's **serverless compute**, not your warehouse. It is billed separately as "Materialized Views" credits.
- **Automatic query rewrite**: if someone queries the *base table* and the MV can answer it, the optimizer may read from the MV silently.

```sql
SHOW MATERIALIZED VIEWS;
ALTER MATERIALIZED VIEW SALES_BY_REGION_MV SUSPEND;   -- stop refreshing
ALTER MATERIALIZED VIEW SALES_BY_REGION_MV RESUME;    -- start again
DROP MATERIALIZED VIEW SALES_BY_REGION_MV;
```

- **Suspended MV**: querying it returns an **error** until you resume. After resume, it catches up on missed changes.
- **Cost control**
  - Use MVs only on slowly changing, frequently read tables.
  - Suspend or drop unused MVs.
  - Track spend in `ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY`.
- **Clustering**: `CLUSTER BY` is possible on an MV for large MVs filtered on a specific column. It adds maintenance cost, so use it only when needed.

---

## Quick Summary

| Point | Remember |
|---|---|
| What | View with stored, auto-refreshed result |
| Why | Fast reads on big tables with repeated aggregations/filters |
| Cost | Storage + serverless refresh credits |
| Limits | Single table, no joins, no `HAVING`, no window functions, no non-deterministic functions |
| Bad fit | Frequently changing tables, small tables, rarely used queries |
| Management | `SHOW`, `SUSPEND`, `RESUME`, `DROP` |
| vs Secure view | MV = speed, Secure view = privacy |
