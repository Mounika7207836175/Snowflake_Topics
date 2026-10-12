# Snowflake: JSON

---

## 1. Syntax and Shapes

### What is JSON?
A **text format** for exchanging data between systems, built from `{ }` objects and `[ ]` arrays.
```json
{"order_id": 101,
 "customer": "Asha",
 "paid": true,
 "discount": null,
 "items": [{"product": "laptop", "qty": 1}]}
```

### The six value types

| Type | Example |
|---|---|
| String | `"Asha"` |
| Number | `101`, `12.5` |
| Boolean | `true`, `false` (lowercase) |
| Null | `null` (lowercase) |
| Object | `{"a": 1}` |
| Array | `[1, 2, 3]` |

**There is no date type.** Dates are strings like `"2025-12-25"`; cast them in Snowflake.

### Syntax rules (usual causes of load errors)
- Keys and text values use **double quotes** only; `'single quotes'` are invalid
- **No trailing comma**: `[1, 2, ]` is invalid
- **No comments**
- Booleans and null are lowercase (`True` or `NULL` are invalid)
- Missing or extra brackets make the record invalid

### Two common file shapes

**Shape A: one record per line (NDJSON / JSON Lines)**
```
{"id": 1, "name": "Asha"}
{"id": 2, "name": "Ravi"}
```
Each line is a record; Snowflake loads each as **one row**.

**Shape B: one big array of records**
```json
[
  {"id": 1, "name": "Asha"},
  {"id": 2, "name": "Ravi"}
]
```
Loaded as is, the **whole array becomes one row** (wasteful, and can hit the 16 MB VARIANT limit).

### The JSON file format
```sql
CREATE OR REPLACE FILE FORMAT JSON_FF
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE;
```
- `TYPE = JSON` tells Snowflake how to read the file
- **`STRIP_OUTER_ARRAY = TRUE`** removes the outer `[ ]` so each record inside becomes its **own row** (use for Shape B; not needed for Shape A)

### Loading it (preview)
```sql
CREATE OR REPLACE TABLE ORDERS_RAW (raw VARIANT);

COPY INTO ORDERS_RAW
FROM @my_stage/orders.json
FILE_FORMAT = (FORMAT_NAME = JSON_FF);
```
The table needs **one `VARIANT` column**; each record lands as one row.

### Why JSON is everywhere
APIs and web apps send it; logs and event streams use it; records can differ in fields and nest naturally.

---

## 2. File Format Options, Errors and Previewing

### Useful options
```sql
CREATE OR REPLACE FILE FORMAT JSON_FF
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE
  STRIP_NULL_VALUES = TRUE
  ALLOW_DUPLICATE = FALSE
  REPLACE_INVALID_CHARACTERS = TRUE;
```

| Option | What it does |
|---|---|
| `STRIP_OUTER_ARRAY` | Removes the outer `[ ]` so each record is its own row |
| `STRIP_NULL_VALUES` | Removes fields or elements whose value is `null` while loading (the whole key is gone) |
| `ALLOW_DUPLICATE` | If `TRUE`, allows repeated keys in one object and keeps the last value; default `FALSE` errors |
| `REPLACE_INVALID_CHARACTERS` | Replaces bad (non-UTF-8) characters instead of failing |
| `COMPRESSION` | Default `AUTO` detects gzip and similar; compressed files load directly |

**Careful with `STRIP_NULL_VALUES`:** the whole key is removed, so afterwards you can't tell "null" from "never sent". It also doesn't remove the need to handle NULLs in queries (a missing key still reads as NULL).

### Preview a file before loading
```sql
SELECT $1 FROM @my_stage/orders.json (FILE_FORMAT => 'JSON_FF');
SELECT $1:order_id, $1:customer FROM @my_stage/orders.json (FILE_FORMAT => 'JSON_FF');
```
`$1` is the whole JSON record, read straight from the stage with no table needed.

### Check for errors without loading
```sql
COPY INTO ORDERS_RAW
FROM @my_stage/orders.json
FILE_FORMAT = (FORMAT_NAME = JSON_FF)
VALIDATION_MODE = RETURN_ERRORS;
```
Lists the problems; loads nothing.

### Bad records: `ON_ERROR`
```sql
COPY INTO ORDERS_RAW
FROM @my_stage/orders.json
FILE_FORMAT = (FORMAT_NAME = JSON_FF)
ON_ERROR = CONTINUE;
```

| Setting | Behavior on a bad record |
|---|---|
| `ABORT_STATEMENT` (default) | Whole load stops; nothing loaded |
| `CONTINUE` | Skips bad rows, loads good ones |
| `SKIP_FILE` | Skips the entire file that has a bad record |

After a load, see rejected rows:
```sql
SELECT * FROM TABLE(VALIDATE(ORDERS_RAW, JOB_ID => '_last'));
```

### Track where each row came from
```sql
COPY INTO ORDERS_RAW (raw, file_name, row_num, load_ts)
FROM (
  SELECT $1, METADATA$FILENAME, METADATA$FILE_ROW_NUMBER, CURRENT_TIMESTAMP()
  FROM @my_stage/orders.json
)
FILE_FORMAT = (FORMAT_NAME = JSON_FF);
```
The table needs those extra columns. Storing source file name and row number helps debugging.

### JSON inside a JSON string (double-encoded)
```json
{"id": 1, "payload": "{\"city\": \"Hyderabad\"}"}
```
`raw:payload` is a **string**, not an object. Parse it first:
```sql
SELECT PARSE_JSON(raw:payload::STRING):city::STRING FROM ORDERS_RAW;
```

### Limits
- One JSON record loaded into `VARIANT` can be about **16 MB** at most
- Each record needs to be valid on its own

---

## 3. Producing JSON, Gotchas and Loading Tips

### PARSE_JSON vs TO_JSON
```sql
SELECT OBJECT_CONSTRUCT(*) FROM CUSTOMERS;           -- each row as an object (VARIANT)
SELECT TO_JSON(OBJECT_CONSTRUCT('a', 1, 'b', 'x'));  -- {"a":1,"b":"x"}  (a plain string)
```
- **`PARSE_JSON`**: JSON **text** to `VARIANT`
- **`TO_JSON`**: `VARIANT` (object or array) to JSON **text**
- They are opposites

### Writing query results out as JSON files
```sql
COPY INTO @my_stage/customers_
FROM (SELECT OBJECT_CONSTRUCT(*) FROM CUSTOMERS)
FILE_FORMAT = (TYPE = JSON);
```
When unloading to JSON, the query must produce **a single VARIANT or OBJECT column**, hence `OBJECT_CONSTRUCT(*)`.

### Key name gotchas
```sql
SELECT raw:order_id           FROM ORDERS_RAW;   -- fine
SELECT raw:"order-id"         FROM ORDERS_RAW;   -- hyphens or spaces need double quotes
SELECT raw:Order_ID           FROM ORDERS_RAW;   -- NULL if the key is order_id (case-sensitive)
```
Keys inside VARIANT are **not** uppercased, unlike column names.

### Different kinds of "empty"

| JSON | Meaning |
|---|---|
| `{"a": null}` | Key present, value null |
| `{}` | Empty object (no keys) |
| `{"a": []}` | Key present, empty array |
| `{"a": ""}` | Key present, empty text |
| key not there | Missing key |

All different. After casting, only the first and the last behave like SQL NULL. Check empty arrays with `ARRAY_SIZE(raw:a) = 0`.

### Numbers: cast to the type you want
```sql
SELECT raw:price::NUMBER(10,2) AS price FROM ORDERS_RAW;
```
For money, always cast to a proper `NUMBER(p,s)`.

### Same field, different types across records
```sql
SELECT TYPEOF(raw:age), COUNT(*) FROM CUSTOMERS_RAW GROUP BY 1;
```
Shows how many records use each type. A safe read is `TRY_TO_NUMBER(raw:age::STRING)`, which returns NULL instead of failing.

### File size tips for loading
- Aim for compressed files of roughly **100 to 250 MB** each so Snowflake can load many in parallel
- One huge file loads slowly; thousands of tiny files add overhead
- Compressed files (like `.gz`) load directly

### JSON gotcha checklist
1. Quotes: double only
2. No trailing commas, no comments
3. Dates are strings; cast them
4. A big array needs `STRIP_OUTER_ARRAY = TRUE`
5. Keys are case-sensitive; odd names need `"quotes"`
6. Null vs missing vs empty are different
7. `STRIP_NULL_VALUES` loses the null-vs-missing difference
8. Mixed types: check with `TYPEOF`, read with `TRY_TO_*`

---

## Quick Lab
```sql
-- Which of these is valid JSON? A and D are valid; B has single quotes; C has a trailing comma
-- A) {"name": "Asha", "age": 28}
-- B) {'name': 'Asha', 'age': 28}
-- C) {"name": "Asha", "age": 28,}
-- D) {"name": "Asha", "paid": true}

SELECT TO_JSON(OBJECT_CONSTRUCT('a', 1, 'b', 'x')) AS json_text,   -- {"a":1,"b":"x"}
       TYPEOF(PARSE_JSON('{"age": "28"}'):age)      AS age_type;    -- VARCHAR
```

---

## Quick Summary

| Point | Remember |
|---|---|
| JSON types | String, number, boolean, null, object, array; **no date type** |
| Syntax | Double quotes only, no trailing commas, no comments, lowercase `true`/`false`/`null` |
| File shapes | One record per line, or one big array |
| `STRIP_OUTER_ARRAY` | Each element of the outer array becomes its own row |
| `STRIP_NULL_VALUES` | Removes null keys at load; loses null vs missing |
| Preview | `SELECT $1 FROM @stage/file (FILE_FORMAT => '...')` |
| Check without loading | `VALIDATION_MODE = RETURN_ERRORS` |
| Bad records | `ON_ERROR`: `ABORT_STATEMENT` (default), `CONTINUE`, `SKIP_FILE` |
| Track source | `METADATA$FILENAME`, `METADATA$FILE_ROW_NUMBER` |
| PARSE_JSON / TO_JSON | Text to VARIANT / VARIANT to text |
| Odd key names | `raw:"order-id"`; keys are case-sensitive |
| Mixed types | `TYPEOF` to inspect, `TRY_TO_*` to read safely |
| Record size | About 16 MB per VARIANT value |
| File size | Roughly 100 to 250 MB compressed per file |
