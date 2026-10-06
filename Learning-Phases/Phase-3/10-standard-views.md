# Snowflake standard views

## Concept

A standard view stores a named SELECT query definition, not a separate persistent copy of the result rows. When you query it, Snowflake resolves the underlying query against its sources, subject to normal query optimization and caching.

In this lab the source is a Snowflake table, not a live connection to MySQL. A view does not inherently connect to another database system.

## Why use one?

- Reuse filters, joins, and calculations across reports.
- Give consumers a simpler set of names and columns.
- Expose selected data through controlled grants instead of granting direct source-table access.

A standard view is not automatically faster than its underlying query. Materialized views store results and have different maintenance and cost behavior; secure views add privacy protections. Those are separate topics.

## Hands-on 1: view over the existing external table

Prerequisite: the earlier holidays external-table lab exists. Select an existing warehouse and a role with the necessary privileges. Execute CREATE statements once; if an object exists, inspect it rather than blindly replacing it.

```sql
-- Choose the schema containing the completed holidays lab.
USE DATABASE LEARNING_DB;
USE SCHEMA EXTERNAL_TABLE_LAB;

-- Save a reusable country filter without copying the external rows.
-- Explicit columns make the view's output interface clear.
CREATE VIEW argentina_holidays_v AS
SELECT holiday_date, holiday_name
FROM LEARNING_DB.EXTERNAL_TABLE_LAB.holidays_external_typed
WHERE country_code = 'AR';

-- Read the saved query and request a predictable result order.
SELECT * FROM argentina_holidays_v ORDER BY holiday_date;
```

A standard view needs no separate refresh. However, its external-table source still needs up-to-date file metadata. New Azure files do not become visible merely because you created a view.

## Hands-on 2: prove that source changes appear

```sql
-- Create an isolated writable source for the demonstration.
CREATE TABLE view_demo_orders (
  order_id INT,
  amount NUMBER(10,2)
);

-- Start with one order so the initial result is easy to verify.
INSERT INTO view_demo_orders VALUES (1, 500.00);

-- Save aggregation logic; do not materialize a second dataset.
CREATE VIEW orders_summary_v AS
SELECT COUNT(*) AS order_count, SUM(amount) AS total_amount
FROM view_demo_orders;

-- Expected: 1 order and total amount 500.
SELECT * FROM orders_summary_v;

-- Change the base table because Snowflake views are not DML targets.
INSERT INTO view_demo_orders VALUES (2, 700.00);

-- Expected: 2 orders and total amount 1200, without refreshing the view.
SELECT * FROM orders_summary_v;

-- Remove only the second order from the source table.
DELETE FROM view_demo_orders WHERE order_id = 2;

-- Expected: 1 order and total amount 500 again.
-- SELECT reads the result; it does not modify the table.
SELECT * FROM orders_summary_v;
```

## Clarifications from the lesson

- “Saved command” is close; the precise description is a saved SELECT query.
- INSERT/UPDATE/DELETE on a base table change that table. Querying a view reads the resulting data.
- Snowflake does not support INSERT, UPDATE, or DELETE directly against these views.
- A view is not a point-in-time copy. CREATE TABLE ... AS SELECT creates an independent copy instead.
- Saying the view does not store result rows does not mean Snowflake cannot cache query results; caching is a separate feature.

## Production considerations

| Concern | Guidance |
|---|---|
| Source dependencies | Dropping or incompatibly altering a source can break its views. |
| SELECT * | Prefer explicit column lists in definitions; source schema changes can cause unexpected failures. |
| Performance | Complex joins and many layers of views can still produce expensive queries. |
| Output order | Use ORDER BY in the consuming query when order matters. |
| Permissions | The view owner needs source access; consumers need view SELECT and appropriate parent database/schema USAGE. They do not necessarily need direct source SELECT. |
| Sensitive data | Standard views are not equivalent to secure views. Review privacy requirements separately. |

```sql
-- OPTIONAL: remove just the demonstration view when it is no longer needed.
-- Its source table and source rows remain unchanged.
DROP VIEW orders_summary_v;
```

## Review

Question: If order 2 is deleted from the source after both inserts, what does the summary view return?

Answer: ORDER_COUNT = 1 and TOTAL_AMOUNT = 500.00. No view refresh is required.


## Sources

- [Snowflake view concepts](https://docs.snowflake.com/en/user-guide/views-introduction)
- [CREATE VIEW reference and restrictions](https://docs.snowflake.com/en/sql-reference/sql/create-view)
