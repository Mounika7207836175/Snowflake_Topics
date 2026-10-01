# Phase 1 | Topic 11: Snowflake SQL Dialect

## What "Dialect" Means
**Dialect** = a database's specific flavor of SQL. All databases use SQL as a base, but each adds its own functions/syntax — like accents of the same language.

Snowflake follows **ANSI SQL** (the standard) closely for basics — `SELECT`, `WHERE`, `JOIN`, `GROUP BY`, `ORDER BY` work as expected. Existing SQL knowledge transfers directly.

**Where it differs:**
1. Querying semi-structured data (JSON) with special operators.
2. Handling Snowflake-specific objects (Stages, Streams, Tasks) via special commands.
3. A few convenience functions other databases lack.

**Interview angle:** *"If someone knows SQL Server, how hard is Snowflake to learn?"* Core querying is nearly identical (both follow ANSI SQL). The curve is mainly Snowflake-specific commands, not relearning basic SQL.

## Distinct Snowflake SQL Features

**1. Colon (`:`) operator — querying JSON in a VARIANT column**
```sql
SELECT raw_data:customer_name, raw_data:amount FROM events_raw;
SELECT raw_data:address:city FROM events_raw;  -- nested JSON
```
Reaches into VARIANT fields directly — something a normal column reference (expecting a single fixed value) can't do. Practiced hands-on in the VARIANT/semi-structured topic.

**2. `QUALIFY` clause — filtering on window functions without a subquery**
```sql
SELECT order_id, customer_id, amount,
  ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
FROM orders
QUALIFY rn = 1;
```
Gets each customer's highest order without wrapping the query in a subquery — cleaner than most other databases require.

**3. `MERGE` statement** — "insert if new, update if existing" logic, central to the **upsert** pattern used heavily in real pipelines (covered fully in incremental loading topic).

**4. `::` casting shorthand**
```sql
SELECT amount::NUMBER(10,2) FROM orders_raw;
```
Shorthand for `CAST(amount AS NUMBER(10,2))` (which still works too).

**Interview angle:** *"Advantage of `QUALIFY` over a subquery + `ROW_NUMBER()`?"* Same result, cleaner/more readable — no need to wrap the query just to filter a window function's output.

---
**Next Topic:** Metadata and system functions
