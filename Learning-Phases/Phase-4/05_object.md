# Snowflake: OBJECT

---

## 1. What, Creating and Reading

### What is an OBJECT?
An `OBJECT` is a collection of **key-value pairs**, like a form with labeled fields.
```json
{"name": "Asha", "age": 28, "city": "Hyderabad"}
```
- **Keys** are always text and can't be NULL
- **Values** can be anything (text, numbers, arrays, another object)
- The curly braces `{ }` in JSON are an `OBJECT`

**OBJECT vs VARIANT:** `VARIANT` can hold anything. `OBJECT` is specifically a `{ key: value }` box. When a VARIANT holds a JSON record, what's inside is an OBJECT (`TYPEOF` shows `OBJECT`).

### Creating an OBJECT
```sql
SELECT OBJECT_CONSTRUCT('name', 'Asha', 'age', 28);   -- {"age": 28, "name": "Asha"}
SELECT {'name': 'Asha', 'age': 28};                   -- same, curly-brace syntax
```
- `OBJECT_CONSTRUCT` takes key, value, key, value... in pairs
- The display may **reorder keys**; don't depend on key order

### A whole table row as an object
```sql
SELECT OBJECT_CONSTRUCT(*) FROM CUSTOMERS;
```
Column names become keys (uppercase unless created with quotes). Handy for sending rows to JSON-based systems or storing them in a VARIANT.

### NULL values: the trap
```sql
SELECT OBJECT_CONSTRUCT('a', 1, 'b', NULL);              -- {"a": 1}   ("b" dropped)
SELECT OBJECT_CONSTRUCT_KEEP_NULL('a', 1, 'b', NULL);    -- {"a": 1, "b": null}
```
`OBJECT_CONSTRUCT` **silently drops** pairs whose value is SQL NULL. Use `OBJECT_CONSTRUCT_KEEP_NULL` to keep the key.

### OBJECT as a column type
```sql
CREATE OR REPLACE TABLE PEOPLE (id NUMBER, info OBJECT);

INSERT INTO PEOPLE
SELECT 1, OBJECT_CONSTRUCT('name', 'Asha', 'age', 28, 'address', {'city': 'Hyderabad'});
```
Unlike VARIANT, this column only accepts objects.

### Reading values
```sql
SELECT info:name                FROM PEOPLE;   -- "Asha"   (colon)
SELECT info['name']             FROM PEOPLE;   -- "Asha"   (square brackets)
SELECT GET(info, 'name')        FROM PEOPLE;   -- "Asha"   (function)
SELECT info:address.city        FROM PEOPLE;   -- "Hyderabad" (deeper with a dot)

SELECT info:name::STRING AS name, info:age::NUMBER AS age FROM PEOPLE;   -- cast for plain values
```
- Keys are **case-sensitive**: `info:Name` returns NULL
- A missing key returns NULL
- `{'x': 5}:x::NUMBER` returns 5 as a number (or use `OBJECT_CONSTRUCT('x', 5):x::NUMBER`)

### Listing the keys
```sql
SELECT OBJECT_KEYS(info) FROM PEOPLE;    -- ["address", "age", "name"]
```
Returns an **array** of key names; useful when you don't know what fields a record has.

---

## 2. Modifying, Combining and Missing Keys

### Objects can't be changed in place
These functions return a **new object**. They never change the table by themselves; you save the result with `UPDATE`.

### OBJECT_INSERT: add or replace a key
```sql
SELECT OBJECT_INSERT({'name': 'Asha'}, 'age', 28);
-- {"age": 28, "name": "Asha"}

SELECT OBJECT_INSERT({'name': 'Asha'}, 'name', 'Mounika');
-- ERROR: the key already exists

SELECT OBJECT_INSERT({'name': 'Asha'}, 'name', 'Mounika', TRUE);
-- {"name": "Mounika"}   (4th argument TRUE = update if it exists)
```
With the 4th argument `FALSE` or omitted, an existing key gives an **error**.

### OBJECT_DELETE: remove keys
```sql
SELECT OBJECT_DELETE({'a': 1, 'b': 2, 'c': 3}, 'b');        -- {"a": 1, "c": 3}
SELECT OBJECT_DELETE({'a': 1, 'b': 2, 'c': 3}, 'a', 'c');   -- {"b": 2}
```

### OBJECT_PICK: keep only the keys you want
```sql
SELECT OBJECT_PICK({'a': 1, 'b': 2, 'c': 3}, 'a', 'c');     -- {"a": 1, "c": 3}
```
Delete removes the listed keys; pick keeps only the listed keys.

### Saving the change into a table
```sql
UPDATE PEOPLE
SET info = OBJECT_INSERT(info, 'age', 29, TRUE)
WHERE id = 1;
```
The function builds the new object; `SET info = ...` stores it.

### OBJECT_AGG: many rows into one object
```sql
-- table SALES_SUMMARY: product, qty
SELECT OBJECT_AGG(product, qty) FROM SALES_SUMMARY;
-- {"laptop": 10, "phone": 25, "tablet": 7}
```
- The key column must be text and not NULL
- Rows whose value is NULL are skipped
- Use with `GROUP BY` for one object per group

### Keys with spaces or special characters
```sql
SELECT info:"first name" FROM PEOPLE;     -- double quotes
SELECT info['first name'] FROM PEOPLE;    -- or square brackets
```

### Checking for a key
```sql
SELECT info:email IS NULL FROM PEOPLE;   -- TRUE if the key is missing (or the value is SQL NULL)
SELECT ARRAY_CONTAINS('email'::VARIANT, OBJECT_KEYS(info)) FROM PEOPLE;   -- TRUE only if the key exists
```
The second form separates "key missing" from "key present with an empty value".

### Handy checks
```sql
SELECT IS_OBJECT(info) FROM PEOPLE;          -- TRUE
SELECT TYPEOF(info:address) FROM PEOPLE;     -- OBJECT (a nested object)
```

---

## Quick Lab
```sql
SELECT OBJECT_CONSTRUCT('a', 1, 'b', NULL),                -- {"a": 1}
       OBJECT_CONSTRUCT_KEEP_NULL('a', 1, 'b', NULL),      -- {"a": 1, "b": null}
       {'x': 5}:x::NUMBER;                                 -- 5

SELECT OBJECT_INSERT({'a': 1}, 'b', 2)       AS added,     -- {"a": 1, "b": 2}
       OBJECT_DELETE({'a': 1, 'b': 2}, 'a')  AS deleted,   -- {"b": 2}
       OBJECT_PICK({'a': 1, 'b': 2}, 'b')    AS picked;    -- {"b": 2}
```

---

## Quick Summary

| Point | Remember |
|---|---|
| OBJECT | Key-value pairs; keys are text and not NULL |
| OBJECT vs VARIANT | VARIANT holds anything; OBJECT is the `{ }` box inside it |
| Create | `OBJECT_CONSTRUCT(k, v, ...)` or `{'k': v}`; `OBJECT_CONSTRUCT(*)` for a whole row |
| NULL values | `OBJECT_CONSTRUCT` drops them; `OBJECT_CONSTRUCT_KEEP_NULL` keeps them |
| Read | `col:key`, `col['key']`, `GET(col, 'key')`; cast for plain values |
| Keys | Case-sensitive; missing key returns NULL; `OBJECT_KEYS` lists them |
| Add / replace | `OBJECT_INSERT(obj, key, value, TRUE)`; without `TRUE`, an existing key errors |
| Remove / keep | `OBJECT_DELETE` removes keys; `OBJECT_PICK` keeps only listed keys |
| Saving changes | Functions return a new object; store it with `UPDATE ... SET` |
| Rows to object | `OBJECT_AGG(key_col, value_col)` |
| Special key names | Use `info:"first name"` or `info['first name']` |
| Key exists? | `ARRAY_CONTAINS(key::VARIANT, OBJECT_KEYS(obj))` |
