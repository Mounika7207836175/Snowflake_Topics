# Snowflake: Stored Procedures

## 1. What and Why

A **stored procedure** is a **saved block of code** inside Snowflake that you run by name with `CALL`.

**Why use it?** A plain SQL statement does one job. A procedure can:
- Run several SQL statements in order
- Use **variables**, **IF/ELSE**, and **loops**
- Handle **errors**
- Be triggered by a **task** so a whole workflow runs on a schedule

**Languages:** SQL (Snowflake Scripting), JavaScript, Python, Java, Scala. We start with SQL.

```sql
CREATE OR REPLACE PROCEDURE SAY_HELLO()
RETURNS STRING
LANGUAGE SQL
AS
BEGIN
  RETURN 'Hello from Snowflake';
END;

CALL SAY_HELLO();
```

**Parts**
- `RETURNS`: data type of the returned value
- `LANGUAGE SQL`: language of the body
- `AS BEGIN ... END`: the body
- `CALL`: the only way to run it (you can't `SELECT` from a procedure)

**With a parameter**
```sql
CREATE OR REPLACE PROCEDURE GREET(name STRING)
RETURNS STRING
LANGUAGE SQL
AS
BEGIN
  RETURN 'Hello ' || name;
END;

CALL GREET('Mounika');
```

**Pipeline fit:** Task -> calls a stored procedure -> procedure runs multiple steps (load, clean, insert).

---

## 2. Variables, Logic and Errors

**Variables**
```sql
CREATE OR REPLACE PROCEDURE ROW_COUNT_DEMO()
RETURNS NUMBER
LANGUAGE SQL
AS
DECLARE
  total NUMBER;
BEGIN
  SELECT COUNT(*) INTO :total FROM ORDERS;
  RETURN total;
END;
```
- `DECLARE` defines variables
- `INTO :total` stores the query result in the variable
- `:` is used when a variable appears inside a SQL statement

**IF / ELSE**
```sql
IF (total = 0) THEN
  RETURN 'No data to process';
ELSE
  RETURN 'Processing ' || total || ' rows';
END IF;
```

**Loops**
```sql
FOR i IN 1 TO 3 DO
  INSERT INTO LOG_TABLE VALUES (:i);
END FOR;
```
Also available: `WHILE`, `REPEAT`.

**Error handling**
```sql
BEGIN
  INSERT INTO TARGET SELECT * FROM STAGING;
  RETURN 'Success';
EXCEPTION
  WHEN OTHER THEN
    RETURN 'Failed: ' || SQLERRM;
END;
```
- If anything in `BEGIN` fails, control jumps to `EXCEPTION`
- `SQLERRM` holds the error message
- `STAGING` = landing table for raw data, `TARGET` = final table (just example table names)

**Dynamic SQL:** `EXECUTE IMMEDIATE` runs a SQL string built in code.

---

## 3. Loops and Cursors

- **Loop** = the repetition ("do this for each row")
- **Cursor** = the pointer that walks through a query result one row at a time

A `FOR` loop over a `SELECT` works **without declaring a cursor**. Snowflake creates one behind the scenes.

```sql
FOR rec IN (SELECT order_id, amount FROM ORDERS) DO
  IF (rec.amount > 10000) THEN
    INSERT INTO HIGH_VALUE_ORDERS VALUES (rec.order_id, rec.amount);
  ELSE
    INSERT INTO NORMAL_ORDERS VALUES (rec.order_id, rec.amount);
  END IF;
END FOR;
```
- The loop body runs once per row
- `rec` holds the current row, `rec.amount` reads a column

**Declared cursor (manual control)**
```sql
DECLARE
  c1 CURSOR FOR SELECT order_id FROM ORDERS WHERE amount > ?;
BEGIN
  OPEN c1 USING (5000);
  FETCH c1 INTO ...;
  CLOSE c1;
END;
```
Use it to pass different parameters, reuse the cursor, or fetch rows manually.

**Caution:** row-by-row loops are **slow on big data**. If one SQL statement can do the job, use that instead.

---

## 4. Rights, Return Types and Management

**Owner**
- The owner is the **role that was active when you ran `CREATE PROCEDURE`**
- It is not set by `EXECUTE AS`

```sql
SELECT CURRENT_ROLE();
SHOW PROCEDURES LIKE 'LOAD_DATA';
```

**OWNER vs CALLER**

| Question | Answered by |
|---|---|
| Who is allowed to call it? | `GRANT USAGE ON PROCEDURE` |
| Whose permissions are used while it runs? | `EXECUTE AS OWNER / CALLER` |

- **`EXECUTE AS OWNER`** (default): runs with the creator's permissions. Callers can use data they can't read directly.
- **`EXECUTE AS CALLER`**: runs with the caller's own permissions. It can never do more than the caller could do alone.

```sql
CREATE OR REPLACE PROCEDURE LOAD_DATA()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS OWNER
AS
BEGIN
  INSERT INTO TARGET SELECT * FROM STAGING;
  RETURN 'Done';
END;
```

**Granting access (same for both modes)**
```sql
GRANT USAGE ON PROCEDURE COUNT_ROWS() TO ROLE ANALYST_ROLE;
```

**Practical lab**
```sql
-- 1. As ACCOUNTADMIN: create roles and grant warehouse/database/schema access
USE ROLE ACCOUNTADMIN;
CREATE ROLE ETL_ROLE;
CREATE ROLE ANALYST_ROLE;
GRANT ROLE ETL_ROLE TO USER <your_username>;
GRANT ROLE ANALYST_ROLE TO USER <your_username>;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ETL_ROLE;
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE ANALYST_ROLE;
GRANT USAGE ON DATABASE MY_DB TO ROLE ETL_ROLE;
GRANT USAGE ON DATABASE MY_DB TO ROLE ANALYST_ROLE;
GRANT ALL ON SCHEMA MY_DB.PUBLIC TO ROLE ETL_ROLE;
GRANT USAGE ON SCHEMA MY_DB.PUBLIC TO ROLE ANALYST_ROLE;

-- 2. As ETL_ROLE: create table and procedure (ETL_ROLE becomes owner)
USE ROLE ETL_ROLE;
USE SCHEMA MY_DB.PUBLIC;
CREATE TABLE SECRET_DATA (id NUMBER);
INSERT INTO SECRET_DATA VALUES (1), (2), (3);

CREATE OR REPLACE PROCEDURE COUNT_ROWS()
RETURNS NUMBER
LANGUAGE SQL
EXECUTE AS OWNER
AS
DECLARE total NUMBER;
BEGIN
  SELECT COUNT(*) INTO :total FROM SECRET_DATA;
  RETURN total;
END;

-- 3. As ANALYST_ROLE: call before grant (fails)
USE ROLE ANALYST_ROLE;
CALL MY_DB.PUBLIC.COUNT_ROWS();

-- 4. As ETL_ROLE: grant access
USE ROLE ETL_ROLE;
GRANT USAGE ON PROCEDURE COUNT_ROWS() TO ROLE ANALYST_ROLE;

-- 5. As ANALYST_ROLE: call again (works, returns 3), table read still fails
USE ROLE ANALYST_ROLE;
CALL MY_DB.PUBLIC.COUNT_ROWS();
SELECT * FROM MY_DB.PUBLIC.SECRET_DATA;
```

Then change the procedure to `EXECUTE AS CALLER` and call as the analyst. It fails until you run `GRANT SELECT ON TABLE SECRET_DATA TO ROLE ANALYST_ROLE;`.

**Returning a table**
```sql
CREATE OR REPLACE PROCEDURE GET_ORDERS()
RETURNS TABLE (order_id NUMBER, amount NUMBER)
LANGUAGE SQL
AS
DECLARE
  res RESULTSET DEFAULT (SELECT order_id, amount FROM ORDERS);
BEGIN
  RETURN TABLE(res);
END;

CALL GET_ORDERS();
```

**Calling from a task**
```sql
CREATE TASK LOAD_TASK
  WAREHOUSE = MY_WH
  SCHEDULE = '60 MINUTE'
AS
  CALL LOAD_DATA();
```

**Managing procedures**
```sql
SHOW PROCEDURES;
DESCRIBE PROCEDURE LOAD_DATA();
DROP PROCEDURE LOAD_DATA();
```
`DESCRIBE` and `DROP` need the parentheses and argument types, like `GREET(STRING)`, because procedures can share a name with different parameters.

---

## Quick Summary

| Point | Remember |
|---|---|
| What | Saved block of code with logic, run by name |
| Run with | `CALL` only, not `SELECT` |
| Languages | SQL, JavaScript, Python, Java, Scala |
| Variables | `DECLARE`, `INTO :var` |
| Logic | `IF / ELSE`, `FOR`, `WHILE`, `REPEAT` |
| Row-by-row | `FOR rec IN (SELECT ...) DO` uses a cursor behind the scenes |
| Errors | `EXCEPTION WHEN OTHER THEN`, `SQLERRM` |
| Owner | Role active at `CREATE PROCEDURE` |
| Who can call | Roles granted `USAGE ON PROCEDURE` |
| OWNER vs CALLER | Whose permissions are used while it runs |
| With tasks | Task runs `CALL proc()` for a full workflow |
| Manage | `SHOW`, `DESCRIBE`, `DROP` (with signature) |
