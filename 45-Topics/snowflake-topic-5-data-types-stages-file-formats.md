# Snowflake Notes: Topic 5, Data Types, Storage and File Formats

## Part A: Data types

### 1. Number
- **NUMBER(precision, scale):** precision = total digits (max 38), scale = digits after the decimal point. `NUMBER(10,2)` holds up to 99999999.99.
- `INT`, `INTEGER`, `BIGINT`, `SMALLINT` are all just `NUMBER(38,0)` (whole numbers). `DECIMAL` and `NUMERIC` are also NUMBER.
- **FLOAT / DOUBLE:** approximate decimal numbers.

*Rule:* use `NUMBER(p,s)` for **money**, never FLOAT. FLOAT is approximate, so small errors creep in (the classic example: 0.1 + 0.2 may not be exactly 0.3). Money must be exact.

### 2. String
- **VARCHAR** (synonyms: `STRING`, `TEXT`): text, up to 16 MB.
- `CHAR` also works as VARCHAR in Snowflake.
- The length you declare (`VARCHAR(100)`) does not change storage or speed. Snowflake stores only the characters actually used. It only rejects text longer than your limit.
- **BINARY** stores raw bytes.

### 3. Date and time
| Type | Stores | Use when |
|---|---|---|
| `DATE` | Year-month-day only | Birthdays, order dates |
| `TIME` | Time of day only | Shift start times |
| `TIMESTAMP_NTZ` | Date + time, **no time zone** (the default for `TIMESTAMP`) | Most pipelines: store the value as-is |
| `TIMESTAMP_LTZ` | Date + time stored in UTC, **shown in each user's session time zone** | Events that should look local to each viewer |
| `TIMESTAMP_TZ` | Date + time **plus the original time zone offset** | When the source time zone must be remembered |

*Scenario:* an order is placed at 10:00 AM in Mumbai and a team in London reads it. With `TIMESTAMP_LTZ`, the Mumbai user sees 10:00 and the London user sees 05:30 (same moment). With `TIMESTAMP_NTZ`, both see the raw 10:00, with no time zone attached.

Useful functions: `CURRENT_DATE()`, `CURRENT_TIMESTAMP()`, `DATEADD(day, 7, order_date)`, `DATEDIFF(day, start_date, end_date)`.

### 4. Boolean
`TRUE`, `FALSE`, or `NULL`. When converting text, Snowflake accepts values like `'true'`, `'yes'`, `'1'` as TRUE and `'false'`, `'no'`, `'0'` as FALSE. For numbers, 0 is FALSE and anything else is TRUE.

### 5. Semi-structured types: VARIANT, ARRAY, OBJECT

**VARIANT** holds *any* value (a number, text, array, or full JSON document), up to 16 MB per value. This is how you load JSON without flattening it first.

**OBJECT** is a set of key-value pairs (like `{"name":"Asha","city":"Pune"}`). Keys are text, values are VARIANT.

**ARRAY** is an ordered list (like `[10, 20, 30]`) and can mix types.

```sql
CREATE TABLE customer_events (id INT, data VARIANT);

INSERT INTO customer_events
SELECT 1, PARSE_JSON('{"name":"Asha","city":"Pune","orders":[101,102,103]}');
```
(Use `INSERT ... SELECT` here, because `PARSE_JSON` doesn't work inside a plain `VALUES` list.)

Reading it:
```sql
SELECT
  data:name::STRING      AS name,
  data:city::STRING      AS city,
  data:orders[0]::INT    AS first_order,
  ARRAY_SIZE(data:orders) AS order_count
FROM customer_events;
```
- `data:name` reaches into the JSON using a colon.
- `::STRING` converts (casts) the VARIANT value to a real type. Without it, text values show with quotes.
- Key names are **case-sensitive**: `data:Name` is not `data:name`.
- Arrays start at index **0**.

Building them:
```sql
SELECT ARRAY_CONSTRUCT(1, 2, 3);
SELECT OBJECT_CONSTRUCT('name', 'Asha', 'city', 'Pune');
```

To turn each array element into its own row, use `FLATTEN`:
```sql
SELECT data:name::STRING AS name, o.value::INT AS order_id
FROM customer_events,
LATERAL FLATTEN(input => data:orders) o;
```
Result: one row per order (Asha 101, Asha 102, Asha 103).

*Scenario:* a mobile app sends event data as JSON, and the fields change as the app grows. Load it into one VARIANT column, and new fields appear without changing the table.

### 6. Geography
`GEOGRAPHY` stores locations on Earth as longitude and latitude (points, lines, polygons). Snowflake also has `GEOMETRY` for flat-plane coordinates.

```sql
SELECT ST_MAKEPOINT(77.5946, 12.9716);   -- longitude FIRST, then latitude (Bengaluru)

SELECT ST_DISTANCE(
  ST_MAKEPOINT(77.5946, 12.9716),
  ST_MAKEPOINT(72.8777, 19.0760)
) / 1000 AS km;                          -- distance in meters, divided to get km
```
`ST_DWITHIN(point_a, point_b, meters)` returns TRUE if two points are within a distance.

*Scenario:* a delivery company stores store and customer locations, then finds every store within 5 km of a customer.

### 7. Sequences
A **sequence** is a separate object that generates unique numbers, often used for IDs.
```sql
CREATE SEQUENCE order_seq START = 1 INCREMENT = 1;

SELECT order_seq.NEXTVAL;                 -- 1, then 2, then 3 ...

CREATE TABLE orders (
  order_id INT DEFAULT order_seq.NEXTVAL,
  item STRING
);
```
- Values are **unique**, but **not guaranteed to be gap-free** or perfectly in order (queries running in parallel can skip numbers).
- A simpler option for a table's own ID column: `order_id INT AUTOINCREMENT`.

---

## Part B: Stages and file formats

### 8. What is a stage?
A **stage** is a place where data **files** wait before loading into a table (or after unloading from one). Think of it as a loading dock.

**Internal stages** keep files **inside Snowflake's storage**. Three kinds:

| | User stage | Table stage | Named internal stage |
|---|---|---|---|
| Reference | `@~` | `@%tablename` | `@my_stage` |
| Created | Automatically, one per user | Automatically, one per table | **You create it** |
| Who can use it | Only that user | Anyone who can use that table, loading only into that table | Anyone you grant access to |
| Best for | Your own quick personal files | Files meant for one specific table | Shared team loading, the usual choice |

**External stages** point to files that stay in **your cloud storage** (Azure Blob/ADLS, S3, or GCS). You create one with the storage URL and a way to log in (best practice: a storage integration, covered in the loading topic).
```sql
CREATE STAGE my_ext_stage
  URL = 'azure://myaccount.blob.core.windows.net/mycontainer/data/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<token>');
```

*Which to pick:* files already living in Azure or S3 from other systems use an **external** stage. Files you receive or upload yourself use a **named internal** stage.

### 9. File format objects
A **file format** tells Snowflake *how to read a file*: delimiter, header, quotes, JSON layout, and so on. It is a reusable object, so you define it once.

Supported formats: CSV, JSON, Avro, ORC, Parquet, XML.

```sql
CREATE FILE FORMAT csv_ff
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', '');

CREATE FILE FORMAT json_ff
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE;
```
- `SKIP_HEADER = 1` skips the first line (column names).
- `FIELD_OPTIONALLY_ENCLOSED_BY = '"'` handles values like `"Pune, India"` that contain a comma.
- `STRIP_OUTER_ARRAY = TRUE` turns a JSON file shaped like `[{...},{...}]` into one row per object.

You can attach a file format to a stage (`CREATE STAGE ... FILE_FORMAT = csv_ff`) or name it when loading.

### 10. The PUT command
`PUT` uploads a file from **your computer** to an **internal stage**.
- It works from **SnowSQL** (or other client tools), **not** from a Snowsight worksheet.
- It automatically **compresses** the file to `.gz` and **encrypts** it.
- It does not work with external stages (there you place files in the cloud storage yourself).

```sql
PUT file://C:\snowflake_lab\employees.csv @my_stage;            -- Windows
PUT file:///Users/asha/snowflake_lab/employees.csv @my_stage;   -- Mac/Linux
LIST @my_stage;                                                  -- see what's inside
```

---

## Quick recap
1. NUMBER for exact values (money), FLOAT is approximate. VARCHAR length doesn't change storage.
2. TIMESTAMP_NTZ = no time zone, LTZ = shown in the viewer's time zone, TZ = remembers the offset.
3. VARIANT holds any JSON. Use `col:key::TYPE` to read it and `FLATTEN` to turn arrays into rows. OBJECT = key-value pairs, ARRAY = list.
4. GEOGRAPHY stores Earth locations, longitude first.
5. Sequences give unique IDs, with possible gaps.
6. Stages: internal (user `@~`, table `@%t`, named `@s`) keep files in Snowflake, external point to your cloud storage.
7. File format objects define how to read files. PUT uploads to internal stages from SnowSQL.

---

## Hands-on: create a named stage and file formats, upload a file with PUT (do later on a laptop)

**Step 1: make a sample file on your laptop** and save it as `employees.csv` in a folder like `C:\snowflake_lab\`:
```
id,name,city,salary
1,Asha,Pune,50000
2,Ravi,Chennai,62000
3,Meena,"Hyderabad, India",58000
```
The third row has a comma inside quotes on purpose, to test your file format.

**Step 2: create the objects** (Snowsight worksheet):
```sql
CREATE DATABASE IF NOT EXISTS stage_lab;
USE SCHEMA stage_lab.public;

CREATE FILE FORMAT csv_ff
  TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE FILE FORMAT json_ff TYPE = JSON STRIP_OUTER_ARRAY = TRUE;

CREATE STAGE my_stage FILE_FORMAT = csv_ff;
```

**Step 3: upload with PUT** (in SnowSQL, after connecting):
```sql
USE SCHEMA stage_lab.public;
PUT file://C:\snowflake_lab\employees.csv @my_stage;
LIST @my_stage;
```
You should see `employees.csv.gz` in the list, which shows the automatic compression.

**Step 4: peek at the staged file** (back in Snowsight):
```sql
SELECT $1, $2, $3, $4 FROM @my_stage (FILE_FORMAT => 'csv_ff');
```
You should see 3 rows, and "Hyderabad, India" should stay in one column. Try the same query without the quote setting to see what breaks.

**Step 5 (JSON version):** save this as `emp.json`, PUT it the same way, and query it with `json_ff`:
```json
[{"id":1,"name":"Asha"},{"id":2,"name":"Ravi"}]
```
```sql
SELECT $1:id::INT, $1:name::STRING FROM @my_stage/emp.json.gz (FILE_FORMAT => 'json_ff');
```

**Clean up:** `DROP DATABASE stage_lab;`
