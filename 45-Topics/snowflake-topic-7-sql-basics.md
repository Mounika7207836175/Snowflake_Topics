# Snowflake Notes: Topic 7, SQL Basics in Snowflake

Most SQL works exactly like standard SQL. This topic covers the basics briefly, then the **Snowflake-specific** features.

## 1. SQL command groups

| Group | Meaning | Commands |
|---|---|---|
| **DDL** | Data Definition: create or change structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `DESCRIBE`, `SHOW` |
| **DML** | Data Manipulation: change data | `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `COPY INTO` |
| **DQL** | Data Query: read data | `SELECT` |
| **DCL** | Data Control: permissions | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control | `BEGIN`, `COMMIT`, `ROLLBACK` |

Snowflake note: DDL statements **commit automatically**. You cannot roll them back.

```sql
CREATE TABLE emp (id INT, name STRING, salary NUMBER(10,2));    -- DDL
INSERT INTO emp VALUES (1, 'Asha', 50000), (2, 'Ravi', 62000);  -- DML
UPDATE emp SET salary = salary + 1000 WHERE id = 1;             -- DML
DELETE FROM emp WHERE id = 2;                                   -- DML
SELECT * FROM emp;                                              -- DQL
```

**MERGE** (upsert): update matching rows, insert new ones, in one statement:
```sql
MERGE INTO emp t
USING new_emp s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.salary = s.salary
WHEN NOT MATCHED THEN INSERT (id, name, salary) VALUES (s.id, s.name, s.salary);
```

## 2. SELECT, WHERE, GROUP BY, HAVING, ORDER BY

```sql
SELECT c_mktsegment, COUNT(*) AS customers, AVG(c_acctbal) AS avg_balance
FROM customer
WHERE c_acctbal > 0
GROUP BY c_mktsegment
HAVING COUNT(*) > 1000
ORDER BY customers DESC
LIMIT 5;
```

**Order in which SQL actually runs:**
`FROM` > `WHERE` > `GROUP BY` > `HAVING` > window functions > `QUALIFY` > `SELECT` > `DISTINCT` > `ORDER BY` > `LIMIT`

- `WHERE` filters **rows** before grouping. `HAVING` filters **groups** after grouping.
- **Text comparison in Snowflake is case-sensitive.** `'Building' = 'BUILDING'` is FALSE. Use `ILIKE` or `UPPER()` for case-insensitive matches.
- **NULL:** `NULL = NULL` is not TRUE. Use `IS NULL` and `IS NOT NULL`.

## 3. Joins

| Join | Returns |
|---|---|
| `INNER JOIN` | Only rows that match in both tables |
| `LEFT JOIN` | All rows from the left table, plus matches from the right (NULL if none) |
| `RIGHT JOIN` | All rows from the right table, plus matches from the left |
| `FULL JOIN` | All rows from both, matched where possible |
| `CROSS JOIN` | Every row of A paired with every row of B |

```sql
SELECT c.c_name, n.n_name
FROM customer c
JOIN nation n ON c.c_nationkey = n.n_nationkey;
```

*Scenario:* "customers who never placed an order" uses a `LEFT JOIN` and keeps rows where the right side is NULL.

## 4. Aggregates
`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.
- `COUNT(*)` counts all rows. `COUNT(column)` counts only **non-NULL** values.
- All aggregates **ignore NULLs** (except `COUNT(*)`).
- `COUNT(DISTINCT col)` counts unique values.
- Snowflake extras: `LISTAGG` (join values into one text), `MEDIAN`, `APPROX_COUNT_DISTINCT` (fast estimate for huge tables).

```sql
SELECT r.r_name,
       LISTAGG(n.n_name, ', ') WITHIN GROUP (ORDER BY n.n_name) AS nations
FROM nation n JOIN region r ON n.n_regionkey = r.r_regionkey
GROUP BY r.r_name;
```

## 5. Subqueries
A query inside another query.
```sql
SELECT o_orderkey, o_totalprice
FROM orders
WHERE o_totalprice > (SELECT AVG(o_totalprice) FROM orders);
```
Also usable in `FROM` (a derived table) and with `IN` / `EXISTS`.

## 6. CTEs
A **CTE** (Common Table Expression) is a named temporary result using `WITH`. It makes long queries readable, like naming each step.
```sql
WITH spend AS (
  SELECT o_custkey, SUM(o_totalprice) AS total
  FROM orders
  GROUP BY o_custkey
)
SELECT c.c_name, s.total
FROM spend s
JOIN customer c ON c.c_custkey = s.o_custkey
ORDER BY s.total DESC
LIMIT 5;
```
You can chain several CTEs separated by commas. They exist only for that one query.

## 7. Snowflake-specific features

| Feature | What it does | Example |
|---|---|---|
| `QUALIFY` | Filters on a **window function** result directly, no subquery needed | `SELECT * FROM orders QUALIFY ROW_NUMBER() OVER (PARTITION BY o_custkey ORDER BY o_orderdate DESC) = 1;` |
| `ILIKE` | Case-insensitive `LIKE` | `WHERE c_mktsegment ILIKE 'auto%'` |
| `SELECT * EXCLUDE (...)` | All columns except some | `SELECT * EXCLUDE (c_comment, c_address) FROM customer;` |
| `SELECT * RENAME (...)` | Rename columns while selecting all | `SELECT * RENAME (c_name AS customer_name) FROM customer;` |
| `GROUP BY ALL` | Groups by every non-aggregate column automatically | `SELECT c_mktsegment, COUNT(*) FROM customer GROUP BY ALL;` |
| `ORDER BY ALL` | Orders by all selected columns | `... ORDER BY ALL;` |
| `IFF(cond, a, b)` | Short IF/ELSE | `IFF(o_totalprice > 200000, 'HIGH', 'LOW')` |
| `ZEROIFNULL`, `NVL`, `COALESCE` | Replace NULL | `ZEROIFNULL(c_acctbal)` |
| `TRY_CAST`, `TRY_TO_NUMBER`, `TRY_TO_DATE` | Convert, but return NULL instead of an error when it fails | `TRY_TO_DATE('2024-13-45')` returns NULL |
| `DATE_TRUNC`, `DATEADD`, `DATEDIFF` | Date math | `DATE_TRUNC('month', o_orderdate)` |
| `CREATE OR REPLACE` | Create, replacing any existing object | `CREATE OR REPLACE TABLE t AS SELECT ...;` |
| Column alias reuse | Use an alias later in the same query | `SELECT price*qty AS total, total*0.18 AS tax FROM ...` |
| `FLATTEN` | Turn arrays into rows (from Topic 5) | |
| `SAMPLE` | Return a random sample of rows | `SELECT * FROM orders SAMPLE (1);` (about 1%) |
| `SHOW` / `DESCRIBE` | Inspect objects | `SHOW TABLES;` `DESCRIBE TABLE customer;` |

**CTAS** (Create Table As Select): build a table from a query:
```sql
CREATE TABLE top_customers AS SELECT * FROM customer WHERE c_acctbal > 9000;
```

---

## Quick recap
1. DDL = structure, DML = data changes, DQL = SELECT. DDL auto-commits.
2. WHERE filters rows, HAVING filters groups. Text comparison is case-sensitive.
3. LEFT JOIN + IS NULL finds missing matches.
4. Aggregates ignore NULLs. `COUNT(*)` vs `COUNT(col)` differ.
5. CTEs make queries readable.
6. Snowflake extras: `QUALIFY`, `ILIKE`, `EXCLUDE`, `GROUP BY ALL`, `IFF`, `TRY_` functions, `LISTAGG`.

---

## Hands-on: 15 queries on Snowflake sample data

Snowflake sample data is read-only, and trial accounts include it. Set your context first:
```sql
USE WAREHOUSE learn_wh;
USE SCHEMA snowflake_sample_data.tpch_sf1;
```
Tables: `CUSTOMER`, `ORDERS`, `LINEITEM`, `NATION`, `REGION`, `PART`, `SUPPLIER`.

| # | Task |
|---|---|
| 1 | Show 10 customers without the comment column |
| 2 | Top 5 customers in the BUILDING segment with balance over 5000, highest first |
| 3 | Count of orders per order status |
| 4 | Customers whose market segment starts with "auto", ignoring case |
| 5 | Number of customers per nation (show the nation name) |
| 6 | Each region with its nations listed in one text value |
| 7 | All orders placed in 1995 |
| 8 | Total order value per month |
| 9 | Customers who have more than 25 orders |
| 10 | Orders whose total price is above the average |
| 11 | Top 5 customers by total spend (use a CTE) |
| 12 | The most recent order of each customer (use QUALIFY) |
| 13 | Customers who never placed an order |
| 14 | Label each order HIGH or LOW at a 200000 threshold |
| 15 | In your own database, copy 100 customers into a new table, raise balances for the BUILDING segment, delete negative balances, then count rows |

### Answers
```sql
-- 1
SELECT * EXCLUDE (c_comment) FROM customer LIMIT 10;

-- 2
SELECT c_name, c_acctbal FROM customer
WHERE c_mktsegment = 'BUILDING' AND c_acctbal > 5000
ORDER BY c_acctbal DESC LIMIT 5;

-- 3
SELECT o_orderstatus, COUNT(*) AS orders FROM orders GROUP BY o_orderstatus;

-- 4
SELECT c_name, c_mktsegment FROM customer WHERE c_mktsegment ILIKE 'auto%' LIMIT 20;

-- 5
SELECT n.n_name, COUNT(*) AS customers
FROM customer c JOIN nation n ON c.c_nationkey = n.n_nationkey
GROUP BY n.n_name ORDER BY customers DESC;

-- 6
SELECT r.r_name, LISTAGG(n.n_name, ', ') WITHIN GROUP (ORDER BY n.n_name) AS nations
FROM nation n JOIN region r ON n.n_regionkey = r.r_regionkey
GROUP BY r.r_name;

-- 7
SELECT o_orderkey, o_orderdate, o_totalprice FROM orders
WHERE o_orderdate BETWEEN '1995-01-01' AND '1995-12-31' LIMIT 50;

-- 8
SELECT DATE_TRUNC('month', o_orderdate) AS month, SUM(o_totalprice) AS revenue
FROM orders GROUP BY 1 ORDER BY 1;

-- 9
SELECT o_custkey, COUNT(*) AS orders FROM orders
GROUP BY o_custkey HAVING COUNT(*) > 25 ORDER BY orders DESC;

-- 10
SELECT o_orderkey, o_totalprice FROM orders
WHERE o_totalprice > (SELECT AVG(o_totalprice) FROM orders) LIMIT 20;

-- 11
WITH spend AS (
  SELECT o_custkey, SUM(o_totalprice) AS total FROM orders GROUP BY o_custkey
)
SELECT c.c_name, s.total FROM spend s
JOIN customer c ON c.c_custkey = s.o_custkey
ORDER BY s.total DESC LIMIT 5;

-- 12
SELECT o_custkey, o_orderkey, o_orderdate FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY o_custkey ORDER BY o_orderdate DESC) = 1;

-- 13
SELECT c.c_custkey, c.c_name
FROM customer c LEFT JOIN orders o ON c.c_custkey = o.o_custkey
WHERE o.o_orderkey IS NULL;

-- 14
SELECT o_orderkey, o_totalprice,
       IFF(o_totalprice > 200000, 'HIGH', 'LOW') AS bucket
FROM orders LIMIT 20;

-- 15 (use your own database, since the sample data is read-only)
CREATE DATABASE IF NOT EXISTS sql_lab;
CREATE OR REPLACE TABLE sql_lab.public.cust_copy AS
  SELECT * FROM snowflake_sample_data.tpch_sf1.customer LIMIT 100;
UPDATE sql_lab.public.cust_copy SET c_acctbal = c_acctbal + 100 WHERE c_mktsegment = 'BUILDING';
DELETE FROM sql_lab.public.cust_copy WHERE c_acctbal < 0;
SELECT COUNT(*) FROM sql_lab.public.cust_copy;
```

### Check yourself
- Query 13 should return rows, because the sample data has customers with no orders.
- Query 3: if you wrote the status in the wrong case in a WHERE filter, you'd get zero rows. Case matters.
- Clean up: `DROP DATABASE sql_lab;`
