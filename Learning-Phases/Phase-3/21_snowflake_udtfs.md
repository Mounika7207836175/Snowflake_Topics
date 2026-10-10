# Snowflake: User-Defined Table Functions (UDTFs)

## 1. What and Why

A **UDTF** is a function that returns a **table** (many rows and columns) instead of a single value.

- **Scalar UDF**: one value in, **one value** out (`ADD_TAX(100)` returns 118)
- **UDTF**: input in, **a set of rows** out

**Why use it?** A reusable, parameterized query, like "all orders above X amount". A scalar UDF can't return that, but a UDTF can.

```sql
CREATE OR REPLACE FUNCTION ORDERS_ABOVE(min_amount NUMBER)
RETURNS TABLE (order_id NUMBER, amount NUMBER)
LANGUAGE SQL
AS
$$
  SELECT order_id, amount
  FROM ORDERS
  WHERE amount > min_amount
$$;
```

**Using it: in the `FROM` clause, wrapped in `TABLE()`**
```sql
SELECT * FROM TABLE(ORDERS_ABOVE(5000));

SELECT * FROM TABLE(ORDERS_ABOVE(5000)) WHERE order_id > 100 ORDER BY amount DESC;
```

**Parts**
- `RETURNS TABLE (...)`: you must list output column names and types
- `$$ ... $$`: for a SQL UDTF, the body is a **single SELECT query**
- `TABLE(function_name(args))`: required wrapper when calling it

| | Scalar UDF | UDTF |
|---|---|---|
| Returns | One value | Rows and columns |
| Used in | `SELECT` list, `WHERE`, `GROUP BY` | `FROM` clause with `TABLE()` |
| Example | `SELECT ADD_TAX(amount)` | `SELECT * FROM TABLE(ORDERS_ABOVE(5000))` |

**Remember:** a scalar UDF is a calculator (one answer). A UDTF is a mini table you can pass inputs to.

---

## 2. One Input Row, Many Output Rows

A SQL UDTF can only wrap a single `SELECT`. To turn one input row into many output rows, use Python or JavaScript.

**Python example: split a sentence into words**
```sql
CREATE OR REPLACE FUNCTION SPLIT_WORDS(sentence STRING)
RETURNS TABLE (word STRING)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'Splitter'
AS
$$
class Splitter:
    def process(self, sentence):
        for w in sentence.split():
            yield (w,)
$$;

SELECT * FROM TABLE(SPLIT_WORDS('learning snowflake is fun'));
```
- `HANDLER` is a **class name**
- `process` runs once per input row
- `yield (w,)` emits **one output row** each time (trailing comma makes a tuple)

**Using it on a whole table (lateral join)**
```sql
SELECT t.ticket_id, w.word
FROM TICKETS t, TABLE(SPLIT_WORDS(t.description)) w;
```
For each ticket row, the function runs once and produces many word rows, paired with that ticket's `ticket_id`. A ticket with 10 words gives 10 rows with the same `ticket_id`.

**JavaScript UDTF parts**
- `processRow`: runs for each input row and writes output rows
- `initialize`: optional setup before processing
- `finalize`: optional cleanup that can write extra rows at the end

**Key points**
- Output columns must match `RETURNS TABLE (...)`
- No `INSERT`, `UPDATE` or `DELETE` inside it
- Python UDTFs are slower than SQL ones; use SQL when a plain `SELECT` is enough

---

## 3. Empty Results, Partitions and Management

**Zero output rows**
- **Comma join**: the input row disappears from the result (like an inner join)
- **LEFT JOIN ... ON TRUE**: the input row stays, with NULLs for the function's columns

```sql
SELECT t.ticket_id, w.word
FROM TICKETS t
LEFT JOIN TABLE(SPLIT_WORDS(t.description)) w ON TRUE;
```

**Working on groups: `PARTITION BY`**
```sql
SELECT *
FROM ORDERS o,
     TABLE(MY_UDTF(o.amount) OVER (PARTITION BY o.customer_id));
```
- `OVER (PARTITION BY ...)` splits rows into groups
- The function handles one group at a time; in Python, `end_partition` runs after the last row of each group, so it can output a summary row
- Without `OVER`, each row is handled alone
- Just know the idea at this stage

**Access and management**
```sql
GRANT USAGE ON FUNCTION ORDERS_ABOVE(NUMBER) TO ROLE ANALYST_ROLE;

SHOW USER FUNCTIONS;
DESCRIBE FUNCTION ORDERS_ABOVE(NUMBER);
DROP FUNCTION ORDERS_ABOVE(NUMBER);
```
Signature (argument types) is required, and UDTFs can be overloaded.

**View vs SQL UDTF**

| | View | SQL UDTF |
|---|---|---|
| Takes input? | No | Yes (parameters) |
| Query with | `SELECT * FROM view` | `SELECT * FROM TABLE(fn(args))` |
| Best for | Fixed query | Same query with changing values |

**Quick lab**
```sql
CREATE OR REPLACE TABLE ORDERS (order_id NUMBER, amount NUMBER);
INSERT INTO ORDERS VALUES (1, 3000), (2, 7000), (3, 12000);

SELECT * FROM TABLE(ORDERS_ABOVE(5000));    -- 2 rows
SELECT * FROM TABLE(ORDERS_ABOVE(10000));   -- 1 row
```

---

## Quick Summary

| Point | Remember |
|---|---|
| What | A function that returns a table |
| Run with | `SELECT * FROM TABLE(fn(args))`, in the `FROM` clause |
| Returns | Declared with `RETURNS TABLE (col type, ...)` |
| SQL UDTF | Single `SELECT` body; like a view with parameters |
| Python UDTF | `HANDLER` is a class, `process` per row, `yield (value,)` emits a row |
| JavaScript UDTF | `processRow`, optional `initialize` and `finalize` |
| One row in, many out | `FROM t, TABLE(fn(t.col))` is a lateral join |
| Zero rows output | Comma join drops the input row; `LEFT JOIN ... ON TRUE` keeps it with NULLs |
| Groups | `OVER (PARTITION BY ...)` passes a whole group to the function |
| Cannot do | `INSERT`, `UPDATE`, `DELETE` |
| Access | `GRANT USAGE ON FUNCTION name(types)` |
| Manage | `SHOW USER FUNCTIONS`, `DESCRIBE`, `DROP` (with signature) |
| vs scalar UDF | Scalar = one value, usable in `SELECT`/`WHERE`; UDTF = rows, usable in `FROM` |
