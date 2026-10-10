# Snowflake: User-Defined Functions (UDFs)

## 1. What and Why

A **UDF** is a function you write yourself and use inside a SQL query, just like built-in functions such as `UPPER()` or `ROUND()`.

**Why use it?** The logic you need may not exist as a built-in function. Write it once, name it, and reuse it in many queries.

```sql
CREATE OR REPLACE FUNCTION ADD_TAX(amount NUMBER)
RETURNS NUMBER
LANGUAGE SQL
AS
$$
  amount * 1.18
$$;

SELECT order_id, amount, ADD_TAX(amount) AS amount_with_tax
FROM ORDERS;
```

**Parts**
- `amount NUMBER`: input parameter and its type
- `RETURNS NUMBER`: type of the returned value
- `LANGUAGE SQL`: language of the body
- `$$ ... $$`: wraps the body (for SQL UDFs, a **single expression** or query)

**Languages:** SQL, JavaScript, Python, Java, Scala.

**Scalar UDF:** takes values in and returns **one value** per call, once for each row. UDTFs (table functions) return many rows and are a separate topic.

### UDF vs Stored Procedure

| | UDF | Stored Procedure |
|---|---|---|
| How it runs | Inside a `SELECT` | `CALL` only |
| Returns | A value for each row | One value or a table, once per call |
| Body | One expression (SQL UDF) | Many steps with IF, loops, errors |
| Typical use | Calculations and transformations on column values | Workflows, automation, multi-step jobs |

**Remember:** a function is a calculator you use *inside* a query. A procedure is a worker you *call* to do a whole job.

---

## 2. Languages, NULLs and Management

**SQL UDF returning a query result**
```sql
CREATE OR REPLACE FUNCTION MAX_ORDER_AMOUNT()
RETURNS NUMBER
LANGUAGE SQL
AS
$$
  SELECT MAX(amount) FROM ORDERS
$$;

SELECT MAX_ORDER_AMOUNT();
```

**JavaScript UDF**
```sql
CREATE OR REPLACE FUNCTION GRADE(score NUMBER)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
  if (SCORE >= 90) return 'A';
  if (SCORE >= 75) return 'B';
  return 'C';
$$;

SELECT name, GRADE(marks) FROM STUDENTS;
```
- Inside a JavaScript UDF, parameter names must be written in **UPPERCASE** (`SCORE`, not `score`)
- JavaScript allows IF, loops and variables
- A JavaScript UDF can use `try/catch` inside its body

**Python UDF**
```sql
CREATE OR REPLACE FUNCTION CLEAN_NAME(n STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'clean'
AS
$$
def clean(n):
    return n.strip().title() if n else None
$$;
```
- `HANDLER` names the Python function to run
- Good for string cleaning and for using libraries

**NULL handling**
```sql
CREATE OR REPLACE FUNCTION ADD_TAX(amount NUMBER)
RETURNS NUMBER
LANGUAGE SQL
RETURNS NULL ON NULL INPUT   -- or CALLED ON NULL INPUT (default)
AS $$ amount * 1.18 $$;
```
- `CALLED ON NULL INPUT` (default): function still runs on NULL input
- `RETURNS NULL ON NULL INPUT`: Snowflake skips running it and returns NULL

**Overloading:** same function name with different parameter types is allowed. This is why `DESCRIBE` and `DROP` need the signature.

**Managing UDFs**
```sql
SHOW USER FUNCTIONS;
DESCRIBE FUNCTION ADD_TAX(NUMBER);
DROP FUNCTION ADD_TAX(NUMBER);
```
`CREATE SECURE FUNCTION` hides the code, similar to secure views.

**Performance tip:** SQL UDFs are usually the fastest. JavaScript and Python UDFs run in a separate engine and are slower on huge data. Use them only when SQL can't express the logic.

---

## 3. If/Else in SQL, Limits and Access

**If/else in a SQL UDF with CASE**
```sql
CREATE OR REPLACE FUNCTION GRADE_SQL(score NUMBER)
RETURNS STRING
LANGUAGE SQL
AS
$$
  CASE
    WHEN score >= 90 THEN 'A'
    WHEN score >= 75 THEN 'B'
    ELSE 'C'
  END
$$;

SELECT GRADE_SQL(82);   -- B
```
This does the same job as the JavaScript version and is the better choice. **Rule:** SQL first, JavaScript only when SQL can't express the logic.

**What a UDF cannot do**
- Run `INSERT`, `UPDATE` or `DELETE`
- Return more than one value per call (scalar UDF)
- Run multi-step workflows or use `CALL`
- Use an `EXCEPTION` block (SQL UDFs have no error handling; JavaScript UDFs can use `try/catch` internally)

**Giving access**
```sql
GRANT USAGE ON FUNCTION GRADE_SQL(NUMBER) TO ROLE ANALYST_ROLE;
```
The grant needs the **signature** (argument types).

**Where UDFs are used in pipelines**
- Inside a view or transformation `SELECT` to clean or standardize columns
- Inside a procedure that loads data
- Inside `WHERE` or `GROUP BY` to filter or group on a calculated value

**Mini lab**
```sql
CREATE OR REPLACE TABLE STUDENTS (name STRING, marks NUMBER);
INSERT INTO STUDENTS VALUES ('Asha', 95), ('Ravi', 80), ('Meena', 60);

SELECT name, marks, GRADE_SQL(marks) AS grade FROM STUDENTS;
```
Expected grades: A, B, C. Then try an `INSERT` inside a UDF body and read the error.

---

## Quick Summary

| Point | Remember |
|---|---|
| What | A function you write, used inside a query |
| Run with | `SELECT` (also `WHERE`, `GROUP BY`), not `CALL` |
| Returns | One value per row (scalar UDF) |
| SQL UDF | One expression or query; use `CASE` for if/else |
| JavaScript UDF | Loops, variables, multi-step logic; parameter names in UPPERCASE |
| Python UDF | Needs `RUNTIME_VERSION` and `HANDLER` |
| NULL handling | `CALLED ON NULL INPUT` (default) vs `RETURNS NULL ON NULL INPUT` |
| Overloading | Same name, different parameter types |
| Cannot do | DML, multi-step workflows |
| Access | `GRANT USAGE ON FUNCTION name(types)` |
| Manage | `SHOW USER FUNCTIONS`, `DESCRIBE`, `DROP` (with signature) |
| UDF vs procedure | UDF = calculator inside a query, procedure = worker you `CALL` |
