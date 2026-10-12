# Snowflake: VARIANT

---

## 1. What and Why

### Structured vs semi-structured data
- **Structured**: fits fixed columns (`name`, `age`, `city`)
- **Semi-structured**: has a shape, but it isn't fixed. Records can have different fields, nested fields, or lists. **JSON** is the most common example:

```json
{"name": "Asha", "age": 28,
 "skills": ["SQL", "Python"],
 "address": {"city": "Hyderabad"}}
```
`skills` is a list and `address` is a box inside the box.

### VARIANT: the "holds anything" column
A `VARIANT` column can store a whole JSON record or any semi-structured value: objects, arrays, numbers, text, booleans, dates and nulls.

Why it's useful:
- Load data **as it arrives**, without designing columns first
- If the source adds a new field, **nothing breaks**
- Query inside it with plain SQL

### Create a table and store JSON
```sql
CREATE OR REPLACE TABLE CUSTOMERS_RAW (raw VARIANT);

INSERT INTO CUSTOMERS_RAW
SELECT PARSE_JSON('{"name":"Asha","age":28,"skills":["SQL","Python"],"address":{"city":"Hyderabad"}}');
```
- `PARSE_JSON` turns JSON **text** into a real `VARIANT`
- Use `INSERT ... SELECT`; `PARSE_JSON` inside a `VALUES (...)` clause fails

### Read it back
```sql
SELECT raw FROM CUSTOMERS_RAW;
SELECT raw:name, raw:address.city FROM CUSTOMERS_RAW;
```
`raw:name` means "go into `raw` and get `name`". The result shows with quotes (`"Asha"`) because it is still a VARIANT. Cast to get a normal value:
```sql
SELECT raw:name::STRING AS name,
       raw:age::NUMBER  AS age
FROM CUSTOMERS_RAW;
```

### Check what is inside
```sql
SELECT TYPEOF(raw:age);       -- INTEGER
SELECT TYPEOF(raw:skills);    -- ARRAY
SELECT TYPEOF(raw:address);   -- OBJECT
```
Each value inside a VARIANT has its own type.

### Key facts
- **Key names are case-sensitive**: `raw:name` works, `raw:Name` returns NULL
- A **missing key** returns NULL, not an error
- Snowflake stores VARIANT efficiently (common fields are split into columns internally and compressed)
- Size limit is about 16 MB per value

---

## 2. Nulls, Bad JSON and Type Checks

### Two kinds of "nothing"
Think of a paper form:
- **JSON null** (`{"a": null}`): the form has a line "Mobile number" and the customer wrote "none". The field exists, its value is null.
- **Missing key** (`{"b": 1}`, asking for `a`): the form has no line for it at all.

```sql
SELECT PARSE_JSON('{"a": null}'):a   AS json_null,      -- shows: null (lowercase, a value inside the VARIANT)
       PARSE_JSON('{"b": 1}'):a      AS missing_key;    -- shows: NULL (a normal SQL NULL)
```

They behave differently in checks:
```sql
SELECT IS_NULL_VALUE(PARSE_JSON('{"a": null}'):a);   -- TRUE (it is a JSON null)
SELECT PARSE_JSON('{"a": null}'):a IS NULL;          -- FALSE (it is not SQL NULL)
SELECT PARSE_JSON('{"b": 1}'):a IS NULL;             -- TRUE  (nothing found)
```

**Fix: cast first, then check.** Both cases become SQL NULL:
```sql
SELECT PARSE_JSON('{"a": null}'):a::STRING IS NULL;  -- TRUE
SELECT PARSE_JSON('{"b": 1}'):a::STRING IS NULL;     -- TRUE
```
Rule: always read JSON values with a cast (`::STRING`, `::NUMBER`).

### Bad JSON text: PARSE_JSON vs TRY_PARSE_JSON
```sql
SELECT PARSE_JSON('{"name": "Asha"');        -- ERROR (missing closing brace)
SELECT TRY_PARSE_JSON('{"name": "Asha"');    -- NULL, no error
```
Same `TRY_` idea as `TRY_TO_DATE`. For messy data, use `TRY_PARSE_JSON` and handle the NULL rows separately.

### VARIANT can hold simple values too
```sql
SELECT TO_VARIANT(123);        -- a number inside a VARIANT
SELECT TO_VARIANT('hello');    -- a string inside a VARIANT
SELECT 'hello'::VARIANT;       -- same thing
```

### Checking the type of a value
```sql
SELECT TYPEOF(raw:age);             -- INTEGER
SELECT IS_INTEGER(raw:age);         -- TRUE
SELECT IS_ARRAY(raw:skills);        -- TRUE
SELECT IS_OBJECT(raw:address);      -- TRUE
SELECT IS_VARCHAR(raw:name);        -- TRUE
```
Handy for catching rows where a field arrives as the wrong type (like `age` as text).

### Dates inside JSON are just text
JSON has no date type, so dates arrive as strings. Cast when reading:
```sql
SELECT raw:order_date::DATE            AS order_date,
       raw:created_at::TIMESTAMP_NTZ   AS created_at
FROM ORDERS_RAW;
```
Without the cast you'd be sorting and comparing text, which breaks date logic.

### Common design: raw column + a few real columns
```sql
CREATE TABLE ORDERS_RAW (
  load_ts  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  raw      VARIANT
);
```
Keep the full record in `raw`, add normal columns for things like load time, and pull needed fields into proper columns later in the clean layer.

---

## Quick Lab
```sql
SELECT raw:name::STRING, raw:Name::STRING, TYPEOF(raw:skills) FROM CUSTOMERS_RAW;
-- Asha, NULL, ARRAY

SELECT IS_NULL_VALUE(PARSE_JSON('{"a": null}'):a),   -- TRUE
       PARSE_JSON('{"a": null}'):a IS NULL,          -- FALSE
       TRY_PARSE_JSON('not json');                   -- NULL
```

---

## Quick Summary

| Point | Remember |
|---|---|
| VARIANT | Holds JSON or any semi-structured value in one column |
| Why | Load without designing columns; new source fields don't break anything |
| PARSE_JSON | JSON text to VARIANT; use `INSERT ... SELECT`, not `VALUES` |
| Reading | `raw:key`, then cast with `::STRING`, `::NUMBER`, `::DATE` |
| Case | Key names are case-sensitive; missing key returns NULL |
| TYPEOF / IS_* | Check the type of a value inside VARIANT |
| JSON null vs missing key | Different things; cast before `IS NULL` and both behave the same |
| TRY_PARSE_JSON | Returns NULL on bad JSON instead of an error |
| Dates in JSON | Arrive as text; cast to `DATE` or `TIMESTAMP` |
| Size | About 16 MB per value |
