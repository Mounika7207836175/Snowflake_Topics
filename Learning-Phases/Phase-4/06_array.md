# Snowflake: ARRAY

---

## 1. What, Creating and Reading

### What is an ARRAY?
An `ARRAY` is an **ordered list** of values. In JSON it is the square brackets `[ ]`.
```json
["SQL", "Python", "Snowflake"]
```
- Each value has a **position number (index)**, and counting **starts at 0**
- Values can be **any type, and can be mixed**: `[1, "a", true, {"x": 1}]`
- An array can contain another array (nested array)

**OBJECT vs ARRAY:** an object finds values by **key name**; an array finds them by **position**.

### Creating an ARRAY
```sql
SELECT ARRAY_CONSTRUCT('SQL', 'Python', 'Snowflake');   -- ["SQL", "Python", "Snowflake"]
SELECT [1, 2, 3];                                       -- square-bracket syntax
SELECT ARRAY_CONSTRUCT(1, 'a', TRUE);                   -- mixed types are fine
SELECT ARRAY_CONSTRUCT();                               -- empty array: []
```

### NULLs: different from OBJECT_CONSTRUCT
```sql
SELECT ARRAY_CONSTRUCT(1, NULL, 3);           -- keeps the NULL in its position
SELECT ARRAY_CONSTRUCT_COMPACT(1, NULL, 3);   -- [1, 3]  (removes NULLs)
```
- `ARRAY_CONSTRUCT` keeps the slot so positions don't shift
- `ARRAY_CONSTRUCT_COMPACT` removes NULLs
- (`OBJECT_CONSTRUCT` drops NULL pairs; arrays keep them by default)

### ARRAY as a column type
```sql
CREATE OR REPLACE TABLE STUDENTS (id NUMBER, skills ARRAY);

INSERT INTO STUDENTS
SELECT 1, ARRAY_CONSTRUCT('SQL', 'Python', 'Snowflake');
```
Use `INSERT ... SELECT`, because array functions don't work inside `VALUES (...)`. The column accepts **only arrays**.

### Reading by position (starts at 0)
```sql
SELECT skills[0] FROM STUDENTS;                 -- "SQL"
SELECT skills[2] FROM STUDENTS;                 -- "Snowflake"
SELECT GET(skills, 1) FROM STUDENTS;            -- "Python"
SELECT skills[0]::STRING FROM STUDENTS;         -- SQL (cast for a plain value)
SELECT skills[10] FROM STUDENTS;                -- NULL (out of range, no error)
```
- Square brackets with a number read an element
- A position that doesn't exist returns **NULL**, not an error
- Negative numbers like `skills[-1]` do **not** work as "last element"

### Counting and finding the last element
```sql
SELECT ARRAY_SIZE(skills) FROM STUDENTS;                   -- 3
SELECT skills[ARRAY_SIZE(skills) - 1] FROM STUDENTS;       -- "Snowflake"
```
The last index is always `size - 1`.

### Arrays inside a VARIANT (JSON)
```sql
-- raw = {"name":"Asha","skills":["SQL","Python"]}
SELECT raw:skills[0]::STRING      FROM CUSTOMERS_RAW;   -- SQL
SELECT ARRAY_SIZE(raw:skills)     FROM CUSTOMERS_RAW;   -- 2
SELECT IS_ARRAY(raw:skills)       FROM CUSTOMERS_RAW;   -- TRUE
```
Use the **colon** to reach the key, then **brackets** for the position.

### Nested arrays
```sql
SELECT [[1, 2], [3, 4]][1][0];    -- 3
```
Read left to right: the second inner array `[3, 4]`, then its first element.

---

## 2. Searching, Slicing and Converting

Adding and removing elements is covered later in "Constructing and modifying arrays and objects".

### ARRAY_CONTAINS: is a value inside?
```sql
SELECT ARRAY_CONTAINS('SQL'::VARIANT, skills) FROM STUDENTS;     -- TRUE
SELECT ARRAY_CONTAINS('Java'::VARIANT, skills) FROM STUDENTS;    -- FALSE
```
- **Value first, array second**
- The value must be cast to `VARIANT`
- Typical use: `WHERE ARRAY_CONTAINS('SQL'::VARIANT, skills)`

### ARRAY_POSITION: where is it?
```sql
SELECT ARRAY_POSITION('Python'::VARIANT, skills) FROM STUDENTS;   -- 1
SELECT ARRAY_POSITION('Java'::VARIANT, skills) FROM STUDENTS;     -- NULL (not found)
```
Returns the index (from 0), or NULL if not found.

### ARRAYS_OVERLAP: do two arrays share anything?
```sql
SELECT ARRAYS_OVERLAP([1, 2, 3], [3, 4, 5]);   -- TRUE
SELECT ARRAYS_OVERLAP([1, 2], [3, 4]);         -- FALSE
```

### ARRAY_SLICE: take a piece
```sql
SELECT ARRAY_SLICE(ARRAY_CONSTRUCT('a', 'b', 'c', 'd'), 0, 2);   -- ["a", "b"]
SELECT ARRAY_SLICE(ARRAY_CONSTRUCT('a', 'b', 'c', 'd'), 1, 3);   -- ["b", "c"]
```
**From** is included, **to** is **not included**.

### ARRAY_TO_STRING: array to text
```sql
SELECT ARRAY_TO_STRING(ARRAY_CONSTRUCT('SQL', 'Python', 'Snowflake'), ', ');
-- SQL, Python, Snowflake
```
NULL elements become empty text.

### SPLIT: text to array
```sql
SELECT SPLIT('SQL,Python,Snowflake', ',');   -- ["SQL", "Python", "Snowflake"]
```
Combine with `TRIM` if pieces have extra spaces.

### ARRAY_AGG: rows to array
```sql
SELECT customer_id, ARRAY_AGG(product) AS products
FROM ORDERS
GROUP BY customer_id;
-- 1 | ["laptop", "mouse"]
-- 2 | ["phone"]

SELECT customer_id,
       ARRAY_AGG(product) WITHIN GROUP (ORDER BY product) AS sorted_products
FROM ORDERS GROUP BY customer_id;
```
- An aggregate function: many rows become one array per group
- Skips NULL values
- `WITHIN GROUP (ORDER BY ...)` controls the order inside the array
- The reverse of `FLATTEN` (array to rows), which comes later

### ARRAY_SORT
```sql
SELECT ARRAY_SORT(ARRAY_CONSTRUCT(3, 1, 2));   -- [1, 2, 3]
```

---

## 3. Arrays of Objects and Set Operations

### Arrays of objects: the most common JSON shape
```json
{"order_id": 1,
 "items": [
   {"product": "laptop", "qty": 1},
   {"product": "mouse",  "qty": 2}
 ]}
```
`items` is an array; each element is an object.

### Reading: position first, then key
```sql
SELECT raw:items[0].product::STRING  FROM ORDERS_RAW;   -- laptop
SELECT raw:items[1].qty::NUMBER      FROM ORDERS_RAW;   -- 2
SELECT ARRAY_SIZE(raw:items)         FROM ORDERS_RAW;   -- 2
```
Read left to right: into `items`, the element at position 0, then its `product` key.

**The limit:** a path points to **one position at a time**. To get every item as its own row, use **FLATTEN** (later topic). Arrays of objects are exactly why FLATTEN exists.

### Removing duplicates and NULLs
```sql
SELECT ARRAY_DISTINCT([1, 1, 2, 2, 3]);        -- [1, 2, 3]
SELECT ARRAY_COMPACT([1, NULL, 3]);            -- [1, 3]
```

### Set operations
```sql
SELECT ARRAY_INTERSECTION([1, 2, 3], [2, 3, 4]);   -- [2, 3]  (in both)
SELECT ARRAY_EXCEPT([1, 2, 3], [2]);               -- [1, 3]  (in the first, not the second)
SELECT ARRAYS_OVERLAP([1, 2, 3], [3, 4]);          -- TRUE    (do they share anything?)
```

### ARRAY_FLATTEN: remove one level of nesting
```sql
SELECT ARRAY_FLATTEN([[1, 2], [3, 4]]);    -- [1, 2, 3, 4]
```
Not the same as the `FLATTEN` table function (array to rows).

### Checking what you have
```sql
SELECT IS_ARRAY(raw:items);        -- TRUE
SELECT TYPEOF(raw:items[0]);       -- OBJECT
SELECT TYPEOF(raw:items);          -- ARRAY
```
Handy when a field is sometimes a single value and sometimes a list.

### ARRAY column vs VARIANT column
- `ARRAY` column accepts **only arrays**
- `VARIANT` column accepts anything and is more flexible
- For raw loading use `VARIANT`; use `ARRAY` when the field is always a list

---

## Quick Lab
```sql
SELECT ARRAY_CONSTRUCT('a', 'b', 'c')[0]::STRING AS first,          -- a
       ARRAY_CONSTRUCT('a', 'b', 'c')[3]         AS out_of_range,   -- NULL
       ARRAY_SIZE(ARRAY_CONSTRUCT('a', 'b', 'c')) AS size;          -- 3

SELECT ARRAY_CONTAINS('SQL'::VARIANT, ARRAY_CONSTRUCT('SQL', 'Python'))       AS has_sql,   -- TRUE
       ARRAY_POSITION('Python'::VARIANT, ARRAY_CONSTRUCT('SQL', 'Python'))    AS pos,       -- 1
       ARRAY_SLICE(ARRAY_CONSTRUCT('a', 'b', 'c', 'd'), 0, 2)                 AS slice,     -- ["a","b"]
       ARRAY_TO_STRING(ARRAY_CONSTRUCT('a', 'b', 'c'), '-')                   AS joined;    -- a-b-c

SELECT ARRAY_DISTINCT([1, 1, 2])                  AS d,   -- [1,2]
       ARRAY_INTERSECTION([1, 2, 3], [2, 3, 4])   AS i,   -- [2,3]
       ARRAY_EXCEPT([1, 2, 3], [2])               AS e,   -- [1,3]
       ARRAY_FLATTEN([[1, 2], [3]])               AS f;   -- [1,2,3]
```

---

## Quick Summary

| Point | Remember |
|---|---|
| ARRAY | Ordered list; index **starts at 0**; mixed types allowed |
| Create | `ARRAY_CONSTRUCT(...)` or `[...]`; `ARRAY_CONSTRUCT_COMPACT` removes NULLs |
| Read | `arr[0]`, `GET(arr, 0)`; out of range returns NULL; no negative index |
| Size / last | `ARRAY_SIZE(arr)`; last element is `arr[ARRAY_SIZE(arr) - 1]` |
| Search | `ARRAY_CONTAINS(value::VARIANT, arr)`, `ARRAY_POSITION` (NULL if not found) |
| Slice | `ARRAY_SLICE(arr, from, to)`; `to` is not included |
| Convert | `ARRAY_TO_STRING(arr, sep)`, `SPLIT(text, sep)` |
| Rows to array | `ARRAY_AGG(col)`, optional `WITHIN GROUP (ORDER BY ...)` |
| Array of objects | `raw:items[1].product::STRING`; all items as rows needs `FLATTEN` |
| Clean up | `ARRAY_DISTINCT`, `ARRAY_COMPACT`, `ARRAY_SORT` |
| Set operations | `ARRAY_INTERSECTION`, `ARRAY_EXCEPT`, `ARRAYS_OVERLAP` |
| ARRAY_FLATTEN | Array of arrays into one array; not the same as `FLATTEN` |
| ARRAY vs VARIANT | ARRAY accepts only arrays; VARIANT accepts anything |
