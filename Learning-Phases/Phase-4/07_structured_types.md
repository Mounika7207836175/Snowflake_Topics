# Snowflake: Structured Types (ARRAY, OBJECT, MAP)

---

## 1. What and Why

### The problem with VARIANT, OBJECT and ARRAY
The values inside them are **untyped**. A `VARIANT` doesn't care whether `age` is a number today and text tomorrow. Flexible for raw loading, but nothing checks the shape and queries need casts everywhere.

### Structured types: same containers, with a declared shape
The structured types are `ARRAY`, `OBJECT` and `MAP`, holding elements or key-value pairs with **specific data types**.

- **VARIANT** = a bag you can put anything in
- **Structured type** = a bag with labeled, shaped pockets

| Type | Meaning | Example |
|---|---|---|
| `ARRAY(VARCHAR)` | List where **every element is text** | `['SQL', 'Python']` |
| `OBJECT(city VARCHAR, pin NUMBER)` | Object with **fixed keys and a type for each** | `{'city': 'Hyderabad', 'pin': 500001}` |
| `MAP(VARCHAR, NUMBER)` | Key-value pairs; **all keys one type, all values one type**; keys not fixed in advance | `{'math': 90, 'sci': 85}` |

- Structured `OBJECT` = a form with **fixed field names**
- `MAP` = a **dictionary** with any number of keys (like marks per subject)

### Using them in a table
```sql
CREATE OR REPLACE TABLE STUDENTS_S (
  id      NUMBER,
  skills  ARRAY(VARCHAR),
  address OBJECT(city VARCHAR, pin NUMBER),
  scores  MAP(VARCHAR, NUMBER)
);
```
Structured types became generally available as typed columns in standard Snowflake tables in July 2025 (per one source; check Snowflake docs for the current status). Before that they were mainly for **Iceberg tables**, where Iceberg's list, struct and map correspond to structured ARRAY, OBJECT and MAP.

### Inserting: cast to the declared type
```sql
INSERT INTO STUDENTS_S
SELECT 1,
       ['SQL', 'Python']::ARRAY(VARCHAR),
       {'city': 'Hyderabad', 'pin': 500001}::OBJECT(city VARCHAR, pin NUMBER),
       {'math': 90, 'sci': 85}::MAP(VARCHAR, NUMBER);
```
A value that doesn't fit the shape (like text in a `NUMBER` slot) gives an **error** instead of being stored.

### Reading values
```sql
SELECT skills[0]       FROM STUDENTS_S;   -- SQL
SELECT address:city    FROM STUDENTS_S;   -- Hyderabad
SELECT scores['math']  FROM STUDENTS_S;   -- 90
```
Values are already typed, so `scores['math']` is a number with no `::NUMBER` cast. Syntax is the same as for VARIANT.

### Why use structured types?
- **Type safety**: wrong types rejected at load time
- **No casting** in every query
- **Required for Iceberg tables** (the shape must be known)
- Good for **clean, curated layers** where the shape is stable

### When to still use VARIANT
- **Raw landing layer**: source shape changes or is unknown
- Messy or inconsistent JSON

**Common pattern:** load raw into `VARIANT`, then convert to structured types (or plain columns) in the clean layer.

### VARIANT column vs `ARRAY(VARCHAR)` column
- `VARIANT` stores any kind of data and needs casts when reading
- `ARRAY(VARCHAR)` fixes both "it's an array" and "every element is text"; no casts needed; wrong element types are rejected

---

## 2. Nesting, Converting and MAP Functions

### Nesting: array of objects with a declared shape
```sql
CREATE OR REPLACE TABLE ORDERS_S (
  order_id NUMBER,
  items    ARRAY(OBJECT(product VARCHAR, qty NUMBER))
);

INSERT INTO ORDERS_S
SELECT 1,
       [{'product': 'laptop', 'qty': 1}, {'product': 'mouse', 'qty': 2}]
         ::ARRAY(OBJECT(product VARCHAR, qty NUMBER));

SELECT items[1].product, items[1].qty FROM ORDERS_S;   -- mouse, 2 (typed, no cast)
```
A structured ARRAY, OBJECT or MAP can be the element type of another, so nesting is allowed.

### Converting semi-structured to structured
Usual flow: raw JSON in `VARIANT`, then cast into a typed shape.
```sql
SELECT raw:skills::ARRAY(VARCHAR)                     AS skills,
       raw:address::OBJECT(city VARCHAR, pin NUMBER)  AS address
FROM CUSTOMERS_RAW;
```
- Casting a semi-structured ARRAY to a structured ARRAY is supported; values that don't fit give an **error**
- Structured to `VARIANT` is also supported with a cast (`address::VARIANT`)

### The rule to remember
You **can't place** a MAP, structured OBJECT or structured ARRAY value **inside** a VARIANT, semi-structured OBJECT or semi-structured ARRAY. Convert one side first.

### Working with a MAP
```sql
SELECT scores['math']                      FROM STUDENTS_S;   -- 90
SELECT MAP_SIZE(scores)                    FROM STUDENTS_S;   -- 2
SELECT MAP_KEYS(scores)                    FROM STUDENTS_S;   -- ["math", "sci"]
SELECT MAP_CONTAINS_KEY('math', scores)    FROM STUDENTS_S;   -- TRUE (key first, map second)
SELECT scores['history']                   FROM STUDENTS_S;   -- NULL (missing key)
```
A missing key returns NULL, not an error.

### Structured OBJECT vs MAP

| Situation | Use |
|---|---|
| Fixed, known fields (`city`, `pin`) | `OBJECT(city VARCHAR, pin NUMBER)` |
| Unknown number of keys, same kind of value (marks per subject, counts per product) | `MAP(VARCHAR, NUMBER)` |

### Limits
- All keys in a `MAP` must be the same type; all values the same type
- In an **Iceberg table** column, you can't use VARIANT, semi-structured ARRAY or semi-structured OBJECT as an ARRAY element or MAP value type
- Structured types give up "anything goes" flexibility: a new source field means a table change

### Where each fits in a pipeline
- **Raw layer**: `VARIANT`
- **Clean layer**: plain columns, or structured types when you keep typed nested data
- **Iceberg tables**: structured types (shape must be declared)

---

## Quick Lab
```sql
SELECT {'math': 90, 'sci': 85}::MAP(VARCHAR, NUMBER)['math'];   -- 90

SELECT {'math': 90, 'sci': 85}::MAP(VARCHAR, NUMBER) AS m,                         -- {"math":90,"sci":85}
       MAP_SIZE({'math': 90, 'sci': 85}::MAP(VARCHAR, NUMBER)) AS size,            -- 2
       {'math': 90, 'sci': 85}::MAP(VARCHAR, NUMBER)['history'] AS missing;        -- NULL
```

---

## Quick Summary

| Point | Remember |
|---|---|
| Structured types | `ARRAY(type)`, `OBJECT(key type, ...)`, `MAP(key_type, value_type)` |
| vs VARIANT | VARIANT = anything, needs casts; structured = declared shape, typed, wrong types rejected |
| Insert | Cast to the declared type (`::ARRAY(VARCHAR)`, `::MAP(VARCHAR, NUMBER)`) |
| Read | Same syntax as VARIANT, but values are already typed |
| OBJECT vs MAP | OBJECT = fixed keys; MAP = any number of keys of one type |
| MAP functions | `MAP_SIZE`, `MAP_KEYS`, `MAP_CONTAINS_KEY(key, map)`; missing key returns NULL |
| Convert | Semi-structured to structured with a cast; mismatches give an error |
| Rule | Structured values can't be placed inside VARIANT / semi-structured OBJECT or ARRAY |
| Iceberg | Needs structured types; no VARIANT inside Iceberg column types |
| Pipeline | Raw layer = VARIANT; clean layer = columns or structured types |
