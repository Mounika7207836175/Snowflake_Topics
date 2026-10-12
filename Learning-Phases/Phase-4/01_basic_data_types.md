# Snowflake: Basic Data Types (Numeric, String, Boolean, Binary)

A **data type** tells Snowflake what kind of value a column holds and how to store it.

---

## 1. Numeric Types

### NUMBER: the main numeric type
```sql
NUMBER(precision, scale)
```
- **Precision** = total digits allowed (max 38)
- **Scale** = digits after the decimal point

| Type | Holds | Example value |
|---|---|---|
| `NUMBER(5,2)` | 3 digits before the point, 2 after | 123.45 |
| `NUMBER(10,0)` | whole numbers up to 10 digits | 1234567890 |
| `NUMBER` (no values given) | same as `NUMBER(38,0)`, whole numbers | 42 |

```sql
CREATE TABLE PRODUCTS (
  product_id NUMBER(10,0),
  price      NUMBER(8,2)
);
```
`price` can hold up to 999999.99.

### Synonyms
- `INT`, `INTEGER`, `BIGINT`, `SMALLINT`, `TINYINT`, `BYTEINT` are all `NUMBER(38,0)`
- `DECIMAL` and `NUMERIC` are `NUMBER`
- The "small" ones do **not** use less storage

### What happens at the limits
```sql
INSERT INTO PRODUCTS VALUES (1, 12.3456);   -- stored as 12.35 (rounded to scale 2)
INSERT INTO PRODUCTS VALUES (2, 9999999);   -- ERROR: too many whole digits for NUMBER(8,2)
```
- Extra decimals are **rounded**
- Too many whole digits is an **error**

### FLOAT: approximate decimals
`FLOAT`, `FLOAT4`, `FLOAT8`, `DOUBLE`, `DOUBLE PRECISION`, `REAL` are all the same 64-bit floating point type.
```sql
SELECT 0.1::FLOAT + 0.2::FLOAT;    -- may show 0.30000000000000004
```

### Which to use
- **Money, quantities, IDs, anything exact**: `NUMBER`
- **Scientific or statistical values where tiny error is fine**: `FLOAT`

---

## 2. String Types

### VARCHAR
```sql
CREATE TABLE CUSTOMERS (
  name  VARCHAR(50),
  city  VARCHAR
);
```
- `VARCHAR(50)` allows up to 50 characters
- `VARCHAR` with no length defaults to 16 MB (the maximum allowed is 128 MB)
- `STRING`, `TEXT`, `CHAR`, `CHARACTER` are all `VARCHAR`
- `CHAR(5)` does **not** pad with spaces

### Does length affect storage?
No. `VARCHAR(10)` and `VARCHAR(1000)` use the **same space** for the same text. The length is only a **safety limit** (a longer value gives an error). Use `VARCHAR` without a length unless you want that limit.

### Writing string values
```sql
SELECT 'Mounika';           -- single quotes
SELECT 'It''s fine';        -- two single quotes to include one
SELECT $$It's fine$$;       -- $$ avoids escaping
```

### Comparisons are case-sensitive
```sql
SELECT 'abc' = 'ABC';       -- FALSE
SELECT 'abc' ILIKE 'ABC';   -- TRUE (ILIKE ignores case)
```

### Empty string vs NULL
```sql
SELECT '' IS NULL;                            -- FALSE
SELECT 'Hi ' || NULL;                         -- NULL
SELECT CONCAT_WS(' ', 'Hi', NULL, 'there');   -- NULL
```
- An empty string is **not** NULL
- `||` returns NULL if any input is NULL
- Snowflake's `CONCAT_WS` also does **not** skip NULLs: any NULL argument gives NULL
- To skip NULLs, wrap columns in `COALESCE(col, '')`

### Common string functions
```sql
UPPER('abc')                  -- ABC
LOWER('ABC')                  -- abc
TRIM('  hi  ')                -- hi
LENGTH('hello')               -- 5
SUBSTR('snowflake', 1, 4)     -- snow (position starts at 1, not 0)
REPLACE('a-b-c', '-', '_')    -- a_b_c
SPLIT_PART('a,b,c', ',', 2)   -- b
```

---

## 3. Boolean

`BOOLEAN` holds `TRUE`, `FALSE` or `NULL`.

```sql
CREATE TABLE USERS (user_id NUMBER, is_active BOOLEAN);
INSERT INTO USERS VALUES (1, TRUE), (2, FALSE), (3, NULL);

SELECT * FROM USERS WHERE is_active;        -- only user 1
SELECT * FROM USERS WHERE NOT is_active;    -- only user 2
SELECT * FROM USERS WHERE is_active IS NULL;  -- only user 3
```
- A boolean column can be used **directly** in `WHERE`
- NULL means **unknown**, so user 3 is neither active nor inactive and appears in neither of the first two queries

### Automatic conversion

| Becomes TRUE | Becomes FALSE |
|---|---|
| `'true'`, `'t'`, `'yes'`, `'y'`, `'on'`, `'1'` | `'false'`, `'f'`, `'no'`, `'n'`, `'off'`, `'0'` |
| any number except 0 | the number 0 |

```sql
SELECT 'yes'::BOOLEAN;    -- TRUE
SELECT 0::BOOLEAN;        -- FALSE
SELECT 5::BOOLEAN;        -- TRUE
```
Handy when a source file has flags like `Y/N` or `1/0`.

### NULL in logic (three-valued logic)
```sql
SELECT NULL AND FALSE;    -- FALSE (one false is enough)
SELECT NULL OR TRUE;      -- TRUE  (one true is enough)
SELECT NULL AND TRUE;     -- NULL  (unknown)
SELECT NULL = NULL;       -- NULL, not TRUE
```
- Check NULL with `IS NULL`, never `= NULL`
- For a NULL-safe comparison, use `IS NOT DISTINCT FROM`

---

## 4. Binary

`BINARY` (also `VARBINARY`) holds raw bytes (default size 8 MB, maximum 64 MB). It is displayed as **hexadecimal**.

```sql
SELECT TO_BINARY('Hello', 'UTF-8');            -- 48656C6C6F
SELECT TO_BINARY('48656C6C6F', 'HEX');         -- same bytes
SELECT TO_VARCHAR(TO_BINARY('48656C6C6F', 'HEX'), 'UTF-8');   -- Hello
```

### Where binary shows up in data engineering
- **Hashes** of rows or keys, for detecting changed records
  ```sql
  SELECT SHA2_BINARY('some value', 256);
  SELECT MD5_BINARY('some value');
  ```
- Raw bytes from source systems (encrypted values, IDs)
- A hash stored as `BINARY` takes **half the space** of the same hash stored as a hex string in `VARCHAR`

---

## Quick Lab
```sql
SELECT 0.1 + 0.2;                                  -- exact: 0.3
SELECT 0.1::FLOAT + 0.2::FLOAT;                    -- approximate
SELECT '' IS NULL, 'Hi' || NULL, CONCAT_WS(' ', 'Hi', NULL, 'there');   -- FALSE, NULL, NULL
SELECT NULL = NULL, NULL IS NULL, 'yes'::BOOLEAN, 0::BOOLEAN;           -- NULL, TRUE, TRUE, FALSE
```

---

## Quick Summary

| Type | Remember |
|---|---|
| NUMBER(p,s) | Exact; max precision 38; extra decimals rounded, too many whole digits = error |
| INT, BIGINT, DECIMAL... | Synonyms of `NUMBER`; no storage savings |
| FLOAT / DOUBLE / REAL | Approximate 64-bit; not for money |
| VARCHAR(n) | `STRING`, `TEXT`, `CHAR` are synonyms; `n` is only a limit, no storage change |
| Strings | Case-sensitive (`ILIKE` ignores case); `''` is not NULL; `\|\|` and `CONCAT_WS` return NULL if any input is NULL |
| BOOLEAN | TRUE / FALSE / NULL; `'yes'`, `'1'`, non-zero numbers become TRUE |
| NULL logic | `NULL = NULL` is NULL; use `IS NULL` or `IS NOT DISTINCT FROM` |
| BINARY | Raw bytes shown as hex (default 8 MB, max 64 MB); used for hashes like `SHA2_BINARY`, `MD5_BINARY` |
