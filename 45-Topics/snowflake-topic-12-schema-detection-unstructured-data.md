# Snowflake Notes: Topic 12, Schema Detection and Unstructured Data

Note: "info schema" in the topic list means **INFER_SCHEMA** (it *infers* a table's columns from a file). It is different from `INFORMATION_SCHEMA`, which holds metadata about your database objects.

## How much to practice
- **Part A (schema detection):** worth practicing. It is used in real pipelines when new Parquet or Avro feeds arrive.
- **Part B (directory tables, file URLs, Snowpark file processing):** awareness is enough, unless your job involves documents, images, or audio.

---

# Part A: Schema detection

## 1. The problem
A Parquet or Avro file arrives with 80 columns. To load it, you'd have to type `CREATE TABLE` with 80 names and types, and guess each type. Self-describing files (Parquet, Avro, ORC) already **carry** their column names and types inside, so Snowflake can read them and tell you.

## 2. INFER_SCHEMA
```sql
CREATE OR REPLACE FILE FORMAT parquet_ff TYPE = PARQUET;

SELECT *
FROM TABLE(INFER_SCHEMA(
  LOCATION    => '@lab_stage/infer/',
  FILE_FORMAT => 'parquet_ff'
));
```
It returns **one row per column** found in the files:

| Column | Meaning |
|---|---|
| `COLUMN_NAME` | Name of the column in the file |
| `TYPE` | The Snowflake type it would use (for example `NUMBER(38, 0)`, `TEXT`, `DATE`, `BOOLEAN`) |
| `NULLABLE` | Whether nulls appear |
| `EXPRESSION` | How to read it in a COPY (for example `$1:emp_id::NUMBER(38, 0)`) |
| `FILENAMES` | Which file(s) it came from |
| `ORDER_ID` | The column's position |

Works for **Parquet, Avro, ORC**, and also JSON, and CSV (CSV needs `PARSE_HEADER = TRUE` in the file format so the first line is used as column names).

Useful options: `IGNORE_CASE => TRUE`, `FILES => ('a.parquet')`, and `MAX_FILE_COUNT` to limit how many files it inspects.

## 3. CREATE TABLE ... USING TEMPLATE
Builds the table **automatically** from what INFER_SCHEMA found:
```sql
CREATE OR REPLACE TABLE emp_auto
USING TEMPLATE (
  SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*)) WITHIN GROUP (ORDER BY order_id)
  FROM TABLE(INFER_SCHEMA(
    LOCATION    => '@lab_stage/infer/',
    FILE_FORMAT => 'parquet_ff',
    IGNORE_CASE => TRUE
  ))
);
```
In plain words: INFER_SCHEMA lists the columns, `OBJECT_CONSTRUCT(*)` packs each column's details into a small object, `ARRAY_AGG` collects them in position order, and `USING TEMPLATE` creates a table from that list.

Then load it with **MATCH_BY_COLUMN_NAME**:
```sql
COPY INTO emp_auto
FROM @lab_stage/infer/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

## 4. The case gotcha (very common in real work)
Spark and other tools usually write **lowercase** column names (`emp_id`). Snowflake normally stores names in **UPPERCASE**.
- Without `IGNORE_CASE => TRUE`, the table gets columns named exactly `"emp_id"` (lowercase, in quotes). From then on, `SELECT emp_id FROM emp_auto` **fails**, because Snowflake looks for `EMP_ID`. You would have to write `"emp_id"` with quotes forever.
- With `IGNORE_CASE => TRUE`, columns become `EMP_ID`, and everything works normally.

Rule of thumb: **always use `IGNORE_CASE => TRUE`** unless you have a reason not to.

## 5. Parquet vs Avro for schema detection
| | Parquet | Avro |
|---|---|---|
| Layout | Columnar | Row-based |
| Schema inside the file | Yes | Yes (in the header) |
| Nested data | Supported, becomes OBJECT/ARRAY columns | Supported, often more deeply nested |
| INFER_SCHEMA works? | Yes | Yes. Only `TYPE = AVRO` changes in the file format |

**Always read the inferred types before accepting them.** The inference can pick a wider type than you want (for example `NUMBER(38,0)` for an ID, or `TIMESTAMP_NTZ` for a date). You can edit the `CREATE TABLE` afterward.

## 6. Schema evolution (a newer feature, good to know)
When a sender adds a new column to the file, a normal load either ignores it or fails. With schema evolution enabled on the table (`ENABLE_SCHEMA_EVOLUTION = TRUE`, plus `MATCH_BY_COLUMN_NAME` in the COPY), Snowflake can **add the new column to the table automatically**. It needs the right privilege on the table, so check Snowflake's docs before using it in a real project.

---

# Part B: Unstructured data

**Unstructured data** means files that aren't rows and columns: PDFs, images, audio, scanned invoices. Snowflake can store them in a **stage** and give you ways to find them, link to them, and process them.

## 7. Directory tables
A **directory table** is a built-in catalog of the files in a stage, queryable with SQL.

Enable it when creating the stage:
```sql
CREATE OR REPLACE STAGE doc_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
```
(`SNOWFLAKE_SSE` encryption is needed for an internal stage that will serve files by URL.)

Refresh it after files change, then query it:
```sql
ALTER STAGE doc_stage REFRESH;

SELECT relative_path, size, last_modified, file_url
FROM DIRECTORY(@doc_stage);
```
| Column | Meaning |
|---|---|
| `RELATIVE_PATH` | File path inside the stage |
| `SIZE` | Bytes |
| `LAST_MODIFIED` | When the file last changed |
| `MD5` / `ETAG` | File fingerprints |
| `FILE_URL` | The permanent Snowflake URL of the file |

Notes:
- For an **internal** stage, run `ALTER STAGE ... REFRESH` after adding files.
- For an **external** stage (Azure), the directory table can refresh automatically when new files land, using cloud event notifications (Azure Event Grid), or you refresh it manually.

*Scenario:* find all PDFs larger than 5 MB uploaded this week:
```sql
SELECT relative_path, size
FROM DIRECTORY(@doc_stage)
WHERE relative_path ILIKE '%.pdf' AND size > 5*1024*1024
  AND last_modified > DATEADD(day, -7, CURRENT_TIMESTAMP());
```

## 8. Three kinds of file URLs

| | Scoped URL | File URL | Pre-signed URL |
|---|---|---|---|
| Function | `BUILD_SCOPED_FILE_URL(@stage, 'path')` | `BUILD_STAGE_FILE_URL(@stage, 'path')` (also the `FILE_URL` column) | `GET_PRESIGNED_URL(@stage, 'path', seconds)` |
| Lifetime | Short (expires, around 24 hours) | Permanent | You choose, in seconds |
| Who can open it | Only the user who made it, with their role's privileges | Anyone with a Snowflake login **and** read privilege on the stage | **Anyone who has the link**, no login |
| Typical use | Passing a file safely to a function or a view | Apps and REST calls inside your organization | Sharing a file outside, like with an auditor |

```sql
SELECT relative_path,
       BUILD_SCOPED_FILE_URL(@doc_stage, relative_path)      AS scoped_url,
       GET_PRESIGNED_URL(@doc_stage, relative_path, 600)     AS presigned_10_min
FROM DIRECTORY(@doc_stage);
```
**How to choose:**
- Showing a file inside a Snowflake app or a view: **scoped**.
- A system inside your company calling Snowflake with its own login: **file URL**.
- Sending a one-off download link to someone with no Snowflake account: **pre-signed**, with a **short expiry**.

**Security point:** a pre-signed URL works for **anyone** who gets the link until it expires. Keep the expiry short (minutes, not days).

## 9. Processing files with Snowpark
**Snowpark** lets you write **Python** that runs **inside Snowflake**. For unstructured files, Python is how you actually open and read the contents (extract text from a PDF, read image size, parse an audio file), since SQL alone can't.

The most common way: a **Python function (UDF)** that opens a file with `SnowflakeFile` and returns something:
```sql
CREATE OR REPLACE FUNCTION file_size_kb(file_url STRING)
RETURNS FLOAT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'run'
AS
$$
from snowflake.snowpark.files import SnowflakeFile

def run(file_url):
    with SnowflakeFile.open(file_url, 'rb') as f:
        return len(f.read()) / 1024
$$;

SELECT relative_path,
       file_size_kb(BUILD_SCOPED_FILE_URL(@doc_stage, relative_path)) AS size_kb
FROM DIRECTORY(@doc_stage);
```
How the pieces fit:
1. The directory table lists the files.
2. `BUILD_SCOPED_FILE_URL` produces a secure link for each file.
3. The Python function opens that link, reads the file, and returns a value.
4. The result comes back as a normal SQL column.

Replace the function body with anything: pull text out of a PDF with a library, count words, or read an image's dimensions. Pick a Python runtime version that your account supports (3.10 or 3.11). Heavy file processing is memory-hungry, so on bigger jobs a Snowpark-optimized warehouse (Topic 6) can help.

*Scenario:* 10,000 invoice PDFs sit in a stage. A Snowpark function extracts invoice number and total from each, and the results land in an ordinary table that finance can query.

---

## Hands-on: auto-create a table from a Parquet file with INFER_SCHEMA

**Step 0: setup**
```sql
USE WAREHOUSE learn_wh;
CREATE DATABASE IF NOT EXISTS practice_db;
CREATE SCHEMA IF NOT EXISTS practice_db.lab;
USE SCHEMA practice_db.lab;
CREATE STAGE IF NOT EXISTS lab_stage;
```

**Step 1: create a Parquet file with lowercase column names** (like a file from Spark)
```sql
COPY INTO @lab_stage/infer/
FROM (
  SELECT * FROM VALUES
    (1,'Asha','Pune',50000.50,'2024-01-05'::DATE,TRUE,'2024-01-05 10:30:00'::TIMESTAMP_NTZ),
    (2,'Ravi','Chennai',62000.00,'2023-11-20'::DATE,TRUE,'2024-02-14 09:15:00'::TIMESTAMP_NTZ),
    (3,'Meena','Hyderabad',58000.75,'2022-07-01'::DATE,FALSE,'2024-03-01 18:45:00'::TIMESTAMP_NTZ)
  AS t("emp_id","name","city","salary","join_date","is_active","last_login")
)
FILE_FORMAT = (TYPE = PARQUET)
HEADER = TRUE
OVERWRITE = TRUE;

LIST @lab_stage/infer/;
```
Expected: one `.parquet` file. (The quotes around the column aliases make the names lowercase.)

**Step 2: ask Snowflake what is inside**
```sql
CREATE OR REPLACE FILE FORMAT parquet_ff TYPE = PARQUET;

SELECT column_name, type, nullable, order_id
FROM TABLE(INFER_SCHEMA(
  LOCATION    => '@lab_stage/infer/',
  FILE_FORMAT => 'parquet_ff'
))
ORDER BY order_id;
```
Expected: **7 rows**, with the lowercase names `emp_id`, `name`, `city`, `salary`, `join_date`, `is_active`, `last_login`, and types like a number type for `emp_id`, `TEXT` for `name` and `city`, a decimal `NUMBER` for `salary`, `DATE`, `BOOLEAN`, and `TIMESTAMP_NTZ`. The exact number sizes may differ slightly.

**Step 3: create the table the "wrong" way** (without IGNORE_CASE)
```sql
CREATE OR REPLACE TABLE emp_lower
USING TEMPLATE (
  SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*)) WITHIN GROUP (ORDER BY order_id)
  FROM TABLE(INFER_SCHEMA(
    LOCATION    => '@lab_stage/infer/',
    FILE_FORMAT => 'parquet_ff'
  ))
);

DESCRIBE TABLE emp_lower;
SELECT emp_id FROM emp_lower;      -- this should FAIL
SELECT "emp_id" FROM emp_lower;    -- this works, but you'd need quotes forever
```
Expected: the `DESCRIBE` shows lowercase column names, the unquoted query gives an "invalid identifier" error, and the quoted one runs (with 0 rows, since nothing is loaded yet).

**Step 4: create it the right way** (with IGNORE_CASE)
```sql
CREATE OR REPLACE TABLE emp_auto
USING TEMPLATE (
  SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*)) WITHIN GROUP (ORDER BY order_id)
  FROM TABLE(INFER_SCHEMA(
    LOCATION    => '@lab_stage/infer/',
    FILE_FORMAT => 'parquet_ff',
    IGNORE_CASE => TRUE
  ))
);

DESCRIBE TABLE emp_auto;
```
Expected: 7 columns with **uppercase** names (`EMP_ID`, `NAME`, ...) and the correct types.

**Step 5: load the data**
```sql
COPY INTO emp_auto
FROM @lab_stage/infer/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT emp_id, name, city, salary, join_date, is_active FROM emp_auto ORDER BY emp_id;
```
Expected: **3 rows**, and the unquoted column names now work. `MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE` connects the file's lowercase `emp_id` to the table's `EMP_ID`.

**Step 6: check the types and adjust if needed**
```sql
CREATE OR REPLACE TABLE emp_final (
  emp_id INT, name STRING, city STRING, salary NUMBER(10,2),
  join_date DATE, is_active BOOLEAN, last_login TIMESTAMP_NTZ
);

COPY INTO emp_final
FROM @lab_stage/infer/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT COUNT(*) FROM emp_final;
```
Expected: **3**. This is the real-life workflow: let INFER_SCHEMA propose the columns, review the types, then write your own final `CREATE TABLE`.

**Optional stretch: directory table and URLs**
```sql
CREATE OR REPLACE STAGE doc_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

COPY INTO @doc_stage/reports/
FROM (SELECT * FROM emp_final)
FILE_FORMAT = (TYPE = CSV COMPRESSION = NONE)
HEADER = TRUE
OVERWRITE = TRUE;

ALTER STAGE doc_stage REFRESH;

SELECT relative_path, size, last_modified FROM DIRECTORY(@doc_stage);

SELECT relative_path,
       BUILD_SCOPED_FILE_URL(@doc_stage, relative_path) AS scoped_url,
       GET_PRESIGNED_URL(@doc_stage, relative_path, 600) AS presigned_10_min
FROM DIRECTORY(@doc_stage);
```
Expected: one file listed. Paste the pre-signed URL into a browser tab within 10 minutes and the file downloads without any login. After 10 minutes it stops working.

**Check yourself**
1. Why did `SELECT emp_id` fail in Step 3? (Without `IGNORE_CASE`, the column was created as lowercase `"emp_id"`, but Snowflake looks for `EMP_ID`.)
2. What does `USING TEMPLATE` do? (It creates a table whose columns come from the list that INFER_SCHEMA produced.)
3. Why use `MATCH_BY_COLUMN_NAME` in the COPY? (It matches file columns to table columns by name, so the order and the letter case don't have to match.)
4. Which URL type would you send to an external auditor? (Pre-signed, with a short expiry.)

---

## Quick recap
1. `INFER_SCHEMA` reads column names and types from Parquet, Avro, ORC (and JSON, or CSV with `PARSE_HEADER`).
2. `CREATE TABLE ... USING TEMPLATE` builds the table from that result. Always add `IGNORE_CASE => TRUE`, then load with `MATCH_BY_COLUMN_NAME`.
3. Review the inferred types, and write your own final `CREATE TABLE` when needed.
4. A directory table is a SQL catalog of the files in a stage. Refresh it after files change.
5. Scoped URL = short-lived and tied to your user. File URL = permanent and needs a login. Pre-signed URL = anyone with the link, for a time you choose.
6. Snowpark Python functions (UDFs) read the contents of unstructured files, through scoped URLs and `SnowflakeFile`.
