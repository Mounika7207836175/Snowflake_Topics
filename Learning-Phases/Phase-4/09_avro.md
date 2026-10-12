# Snowflake: Avro

---

## 1. What It Is and How It Differs from JSON

### What is Avro?
Avro is a **binary file format** made for moving data between systems, especially in **streaming pipelines** (for example Kafka, a popular tool that moves streams of events between systems; you don't need to learn Kafka now). Think of it as JSON's compact, strictly-typed cousin.

### The big idea: the schema travels inside the file
Every Avro file starts with a **schema** (written in JSON) that describes the fields and types, followed by the data in compact binary form. The schema is stored in the **header (start) of the file**.
```json
{"type": "record", "name": "Customer",
 "fields": [
   {"name": "id",   "type": "int"},
   {"name": "name", "type": "string"},
   {"name": "city", "type": ["null", "string"]}
 ]}
```
- `int` and `string` are **declared types**
- `["null", "string"]` means the field is **optional**: a string or null (this is how Avro says "nullable")
- Anyone reading the file knows the exact shape without guessing

### Avro vs JSON

| | JSON | Avro |
|---|---|---|
| Format | Text, human-readable | **Binary**, not readable in a text editor |
| Schema | None (each record describes itself) | **Stored once in the file header** |
| Types | Loose (no dates, one number type) | Strict (int, long, float, double, string, boolean...) |
| File size | Larger (field names repeat on every record) | **Much smaller** (names are stored once) |
| Best for | APIs, flexible data | Streaming, high-volume data transfer |

### Row-based storage
Avro stores data **record by record (row by row)**:
- Great for **writing** streams of events one at a time (stream processing needs quick writes)
- Not ideal for analytics that read a few columns across millions of rows; **column-based** formats (Parquet, ORC) are better for that

### Loading Avro into Snowflake
```sql
CREATE OR REPLACE FILE FORMAT AVRO_FF TYPE = AVRO;

CREATE OR REPLACE TABLE CUSTOMERS_RAW (raw VARIANT);

COPY INTO CUSTOMERS_RAW
FROM @my_stage/customers.avro
FILE_FORMAT = (FORMAT_NAME = AVRO_FF);
```
- Each Avro **record becomes one row**, stored in a `VARIANT` column
- You read it exactly like JSON: `raw:name::STRING`
- You **don't** need `STRIP_OUTER_ARRAY` (Avro isn't an array of text records)

### Preview before loading
```sql
SELECT $1 FROM @my_stage/customers.avro (FILE_FORMAT => 'AVRO_FF');
SELECT $1:id::NUMBER, $1:name::STRING FROM @my_stage/customers.avro (FILE_FORMAT => 'AVRO_FF');
```
The data shows as JSON-like values even though the file is binary.

### Compression
Avro compresses its data blocks itself (for example Snappy or Deflate). Leave Snowflake's `COMPRESSION` option at the default (`AUTO`).

---

## 2. Structure Inside Avro and Loading It Properly

### Avro data types

| Group | Types |
|---|---|
| Simple | `null`, `boolean`, `int`, `long`, `float`, `double`, `bytes`, `string` |
| Complex | `record`, `enum`, `array`, `map`, `union`, `fixed` |

- `record` = group of named fields (like a JSON object)
- `array` = a list
- `map` = key-value pairs
- `enum` = a value from a fixed list (like `"RED"`, `"GREEN"`)
- `union` = "one of these types", mostly used for optional fields: `["null", "string"]`

### How they look in Snowflake after loading
All arrive inside the `VARIANT`:
- `record` becomes an **object**
- `array` becomes an **array**
- `map` becomes an **object**
- `union` with null becomes the **value or null**

Read nested data exactly like JSON:
```sql
SELECT raw:id::NUMBER,
       raw:name::STRING,
       raw:address.city::STRING,       -- nested record
       raw:phones[0]::STRING           -- first item of an array
FROM CUSTOMERS_RAW;
```

### Dates and timestamps (logical types)
Avro stores dates and timestamps as **numbers with a label** (logical types), such as "days since 1970" or "milliseconds since 1970". After loading, **preview the values** and check them with `TYPEOF`. If you see plain numbers where you expect dates, convert them:
```sql
SELECT TO_TIMESTAMP(raw:created_at::NUMBER, 3) FROM CUSTOMERS_RAW;   -- 3 = milliseconds
SELECT DATEADD('day', raw:birth_date::NUMBER, '1970-01-01'::DATE) FROM CUSTOMERS_RAW;
```
Same epoch idea as in the timestamps topic.

### Two ways to load Avro into normal columns

**Option A: pick fields in a `SELECT` while loading**
```sql
COPY INTO CUSTOMERS
FROM (
  SELECT $1:id::NUMBER, $1:name::STRING, $1:city::STRING
  FROM @my_stage/customers.avro
)
FILE_FORMAT = (TYPE = AVRO);
```

**Option B: match field names to column names automatically**
```sql
COPY INTO CUSTOMERS
FROM @my_stage/customers.avro
FILE_FORMAT = (TYPE = AVRO)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```
- Works for Avro, Parquet and ORC (and JSON)
- Fields with no matching column are ignored; columns with no matching field get NULL

### Avro file format options
```sql
CREATE OR REPLACE FILE FORMAT AVRO_FF
  TYPE = AVRO
  COMPRESSION = AUTO
  TRIM_SPACE = FALSE
  REPLACE_INVALID_CHARACTERS = FALSE
  NULL_IF = ('\\N');
```
You rarely need to change these: Avro is already compressed internally and the schema is in the file, so there are far fewer options than for CSV.

### Schema changes in Avro (the idea only)
- Avro was designed for **schema evolution**: a new version of a record can add a field **with a default value**, and old and new files can still be read together
- In Snowflake, a new field simply appears as a **new key in the VARIANT**, so a raw `VARIANT` table doesn't break
- Updating a **typed table** automatically for new fields is the "Schema evolution" topic later in Phase 4

### Unloading: Snowflake can't write Avro
`COPY INTO <location>` can unload to **CSV, JSON and Parquet**, but **not Avro** (nor ORC). In Snowflake, Avro is for **reading**.

### When to choose Avro
- Your **source system** (like a streaming pipeline) already produces it
- You need a **strict, self-describing** format that is smaller than JSON
- Not for analytics storage: Parquet is the better choice there

---

## 3. Practical Handling and the Bigger Picture

### Typical pipeline flow
```
Source / streaming system -> Avro files in cloud storage -> Stage -> COPY INTO (VARIANT raw table)
-> convert to typed columns -> clean table
```
Keep the raw `VARIANT` table as the landing layer, then build the typed table from it.

### Convert raw Avro to typed columns after loading
```sql
CREATE OR REPLACE TABLE CUSTOMERS_CLEAN AS
SELECT raw:id::NUMBER            AS id,
       raw:name::STRING          AS name,
       raw:address.city::STRING  AS city
FROM CUSTOMERS_RAW;
```
Querying the raw `VARIANT` every time works, but a typed table is **easier to use** and avoids repeating casts.

### Arrays of records need FLATTEN
If an Avro record holds an array of records (for example `items`), one path can't read all the items; you use `FLATTEN` (a separate topic coming later in this phase). Same rule as for JSON arrays.

### Peek at the schema Snowflake finds in the file (preview)
```sql
SELECT * FROM TABLE(
  INFER_SCHEMA(LOCATION => '@my_stage/customers.avro', FILE_FORMAT => 'AVRO_FF')
);
```
It lists the column names and types found in the file. Covered properly in the "Schema detection" topic.

### Common problems (concepts)
- **Not a valid Avro file** (truncated or corrupted): the load fails for that file. Check with the preview or `VALIDATION_MODE = RETURN_ERRORS`.
- **Date columns arrive as numbers**: convert them (see logical types above)
- **Different schema versions in one load**: fine into a `VARIANT` table; a typed table needs the fields to match or `MATCH_BY_COLUMN_NAME`
- **Can't open it in a text editor**: it's binary; use the Snowflake preview to inspect it
- **Record too large**: each record loaded into `VARIANT` has the ~16 MB limit

### Format comparison

| | JSON | Avro | Parquet / ORC |
|---|---|---|---|
| Storage | Text | Binary, row-based | Binary, column-based |
| Schema in file | No | Yes | Yes |
| Size | Largest | Smaller | Smallest (great compression) |
| Best for | APIs, flexible data | Streaming and data transfer | Analytics and warehousing |
| Snowflake load | Yes | Yes | Yes |
| Snowflake unload | Yes | **No** | Parquet yes; ORC no |

---

## Quick Lab
A source sends Avro with a nested record `address` and an array `phones`. Paths to read the city and the second phone number:
```sql
SELECT raw:address.city::STRING AS city,
       raw:phones[1]::STRING    AS second_phone     -- arrays start at 0
FROM CUSTOMERS_RAW;
```

---

## Quick Summary

| Point | Remember |
|---|---|
| Avro | Binary, row-based format; common in streaming pipelines |
| Schema | Stored once in the **file header**; self-describing |
| vs JSON | Smaller (names stored once), strictly typed, not human-readable |
| Optional field | Union with null: `["null", "string"]` |
| Row-based | Good for writing event streams; not for analytics (use Parquet/ORC) |
| Load | `TYPE = AVRO`, each record becomes one `VARIANT` row; no `STRIP_OUTER_ARRAY` needed |
| Nested data | Read like JSON: `raw:address.city`, `raw:phones[0]` |
| Dates | Logical types may arrive as numbers; preview and convert (epoch math) |
| Typed table loading | `SELECT` transform inside `COPY INTO`, or `MATCH_BY_COLUMN_NAME` |
| Unload | Snowflake **cannot** unload to Avro (only CSV, JSON, Parquet) |
| Schema evolution | New fields show up as new VARIANT keys; typed-table evolution comes later |
| Arrays of records | Need `FLATTEN` (later topic) |
| Inspect a file | Snowflake preview (`$1`), or `INFER_SCHEMA` for the schema |
