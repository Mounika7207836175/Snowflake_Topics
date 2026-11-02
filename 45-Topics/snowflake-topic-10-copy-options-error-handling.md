# Snowflake Notes: Topic 10, COPY INTO Options and Error Handling

## Test file (used in all examples)
A file with 5 lines for the table `emp_bad (id INT, name STRING, city STRING, salary NUMBER)`:

| Line | Content | Problem |
|---|---|---|
| 1 | `1,Asha,Pune,50000` | Good |
| 2 | `2,Ravi,Chennai,abc` | Salary is text, not a number |
| 3 | `3,Meena,Hyderabad,58000,EXTRA` | 5 values, the table has 4 columns |
| 4 | `4,Kiran,Delhi` | Only 3 values, the table has 4 columns |
| 5 | `5,Divya,Mumbai,70000` | Good |

2 good rows, 3 bad rows.

## 1. ON_ERROR: what to do when a row is bad

| Option | What happens with our file |
|---|---|
| `ABORT_STATEMENT` (**default**) | Stop at the first error. **Nothing** is loaded |
| `CONTINUE` | Skip the 3 bad rows, load the 2 good ones. Status `PARTIALLY_LOADED` |
| `SKIP_FILE` | Skip the **whole file** if it has any bad row. 0 rows loaded |
| `SKIP_FILE_2` | Skip the file if it has **2 or more** bad rows. We have 3, so skipped |
| `SKIP_FILE_4` | Skip only if **4 or more** bad rows. We have 3, so it loads the 2 good rows |
| `SKIP_FILE_50%` | Skip if **50% or more** rows are bad. 3 of 5 is 60%, so skipped |

**Which to choose (real scenarios):**
- Finance or billing data, where every row matters: `ABORT_STATEMENT`.
- Website click logs, where a few broken rows are acceptable: `CONTINUE`.
- Files from an unreliable partner, where one bad row suggests the whole file is wrong: `SKIP_FILE` or `SKIP_FILE_n%`.

**Important:** after `CONTINUE`, the file counts as **loaded**. The skipped rows are **not retried automatically**. You must find them, fix them, and load them yourself (that is what VALIDATE is for).

## 2. VALIDATION_MODE: a dry run

Checks the file **without loading anything**.
```sql
COPY INTO emp_bad FROM @lab_stage/messy/
FILE_FORMAT = (FORMAT_NAME = 'csv_nh')
VALIDATION_MODE = RETURN_ERRORS;
```
| Value | Returns |
|---|---|
| `RETURN_ERRORS` | All errors found in the files |
| `RETURN_ALL_ERRORS` | Same, plus errors from files that were partially loaded earlier |
| `RETURN_5_ROWS` (any number) | The first 5 rows as they would be loaded, so you can eyeball them |

Limit: VALIDATION_MODE **does not work** when the COPY transforms data (section 7).

**Scenario:** before the nightly load of a big file, run `RETURN_ERRORS`. If it returns nothing, run the real load.

## 3. VALIDATE: find the rows after a load

VALIDATION_MODE checks **before** loading. `VALIDATE` checks **after** a load that used `ON_ERROR = CONTINUE`.

```sql
SELECT error, line, rejected_record
FROM TABLE(VALIDATE(emp_bad, JOB_ID => '_last'));
```
- `JOB_ID => '_last'` means "the most recent COPY in this session". You can also pass a query ID from an earlier load.
- It returns the error message, the line number, and `REJECTED_RECORD` (the **raw bad line**).

**Quarantine pattern (very common in real pipelines):** save rejected rows in a table, so someone can fix them later.
```sql
CREATE OR REPLACE TABLE emp_rejected AS
SELECT rejected_record, error, CURRENT_TIMESTAMP() AS captured_at
FROM TABLE(VALIDATE(emp_bad, JOB_ID => '_last'));
```

## 4. Load metadata, the 64-day history, and duplicate prevention

Snowflake keeps a **load history per table** and remembers which files were loaded into it.

- Run the same COPY again: **0 files processed**. This is **duplicate prevention**.
- The memory lasts **64 days**.
- It is tracked **per table**. The same file can be loaded into a *different* table without any problem.
- If a file's contents change, Snowflake usually treats it as a new file (it tracks more than just the name).

**FORCE = TRUE** ignores the history and loads the file again (you get duplicates).

**The 64-day catch:** for a file that was last modified **more than 64 days ago**, Snowflake can't be sure whether it was loaded (the history has expired), so by default it **skips** it. To load such files, add `LOAD_UNCERTAIN_FILES = TRUE` (or `FORCE = TRUE`).

**See the history:**
```sql
SELECT file_name, status, row_count, error_count, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'EMP_BAD',
  START_TIME => DATEADD(hour, -1, CURRENT_TIMESTAMP())));
```

**Reset tip:** `TRUNCATE TABLE` also clears that table's load history, so the same files can be loaded again.

## 5. MATCH_BY_COLUMN_NAME

Loads by **column name** instead of **column position**.
```sql
COPY INTO sales_p FROM @lab_stage/parq/
FILE_FORMAT = (TYPE = PARQUET)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```
- `CASE_INSENSITIVE`: `EMP` matches `emp`. `CASE_SENSITIVE`: must match exactly. `NONE`: off.
- Works for Parquet, Avro, ORC, and JSON. For CSV it works only when the file format has `PARSE_HEADER = TRUE` (uses the header line as column names).
- **Benefit:** if the sender reorders the columns, the load still lands in the right columns. Positional loading would silently put data in the wrong columns.

## 6. PURGE

`PURGE = TRUE` deletes each file from the stage **after it loaded successfully**.
- Files that failed or were skipped stay, so you can investigate them.
- Without PURGE, old files pile up (and you pay for the storage in an internal stage).

## 7. Transformations during load

A `SELECT` inside `COPY INTO` lets you reshape data while loading. Columns are referenced by position: `$1`, `$2`, ...

```sql
COPY INTO emp_audit (id, name, city, salary, source_file, loaded_at)
FROM (
  SELECT $1::INT,
         UPPER($2),
         $3,
         $4::NUMBER,
         METADATA$FILENAME,
         CURRENT_TIMESTAMP()
  FROM @lab_stage/emp/
)
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');
```
- Casts types, changes text case, picks and reorders columns.
- `METADATA$FILENAME` records **which file each row came from** (great for tracing problems). `METADATA$FILE_ROW_NUMBER` records the row number inside the file.
- `CURRENT_TIMESTAMP()` stamps the load time.

**Limits:** `VALIDATION_MODE` and `VALIDATE` do not work with COPY statements that contain a SELECT transformation. `ON_ERROR` still works.

## 8. INSERT and MERGE loads

**COPY only appends rows.** It can't update an existing row. For "new customers plus changed customers" files, the standard pipeline is:

1. `COPY INTO` a **staging table** (emptied before each run).
2. `MERGE` from staging into the **final table**.

```sql
CREATE OR REPLACE TABLE customers (id INT, name STRING, city STRING);
INSERT INTO customers VALUES (1,'Asha','Pune'),(2,'Ravi','Chennai'),(3,'Meena','Hyderabad');

CREATE OR REPLACE TABLE customers_stg (id INT, name STRING, city STRING);
INSERT INTO customers_stg VALUES (2,'Ravi','Bengaluru'),(4,'Kiran','Delhi');
-- in a real pipeline: TRUNCATE customers_stg; COPY INTO customers_stg FROM @stage;

MERGE INTO customers t
USING customers_stg s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.city = s.city
WHEN NOT MATCHED THEN INSERT (id, name, city) VALUES (s.id, s.name, s.city);

SELECT * FROM customers ORDER BY id;
```
**Result:** 4 rows. Ravi's city is updated to Bengaluru (1 row updated), Kiran is added (1 row inserted).

| Method | Duplicate protection |
|---|---|
| `COPY INTO` | **Yes**, through load metadata |
| Plain `INSERT ... SELECT` from a file | **No**, running it twice doubles the rows |
| `MERGE` | Safe to re-run, because the same input gives the same final result (as long as the staging data has one row per id) |

**Gotcha:** if the staging table has **two rows with the same id**, `MERGE` fails with a "duplicate row detected" style error. Fix it by keeping one row per id first:
```sql
SELECT * FROM customers_stg
QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY id) = 1;
```

---

## Quick recap
1. `ON_ERROR`: `ABORT_STATEMENT` default, `CONTINUE` skips bad rows, `SKIP_FILE` and `SKIP_FILE_n` reject whole files.
2. `VALIDATION_MODE` = dry run before loading. `VALIDATE` = inspect rejected rows after loading.
3. Load history is per table, lasts 64 days, and prevents duplicates. `FORCE` overrides it. Older files need `LOAD_UNCERTAIN_FILES`.
4. `MATCH_BY_COLUMN_NAME` loads by name, so reordered columns don't break the load.
5. `PURGE` deletes successfully loaded files from the stage.
6. A `SELECT` inside COPY lets you cast, reorder, and add `METADATA$FILENAME` and a load timestamp.
7. COPY appends only. For updates, COPY into a staging table, then `MERGE`.

---

## Hands-on: load a file with bad rows, use ON_ERROR = CONTINUE, find them with VALIDATE

**Step 0: setup**
```sql
USE WAREHOUSE learn_wh;
CREATE DATABASE IF NOT EXISTS practice_db;
CREATE SCHEMA IF NOT EXISTS practice_db.lab;
USE SCHEMA practice_db.lab;
CREATE STAGE IF NOT EXISTS lab_stage;

CREATE OR REPLACE FILE FORMAT csv_nh
  TYPE = CSV FIELD_DELIMITER = ',' FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE OR REPLACE TABLE emp_bad (id INT, name STRING, city STRING, salary NUMBER);
```

**Step 1: create the messy file inside the stage**
```sql
COPY INTO @lab_stage/messy/
FROM (
  SELECT * FROM VALUES
    ('1,Asha,Pune,50000'),
    ('2,Ravi,Chennai,abc'),
    ('3,Meena,Hyderabad,58000,EXTRA'),
    ('4,Kiran,Delhi'),
    ('5,Divya,Mumbai,70000')
  AS t(line)
)
FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = NONE COMPRESSION = GZIP)
HEADER = FALSE
OVERWRITE = TRUE;

LIST @lab_stage/messy/;
```
Expected: one `.csv.gz` file. (If this step errors, create the 5 lines in Notepad as `messy.csv` and upload it into `LAB_STAGE/messy/` with the **+ Files** button in Snowsight.)

**Step 2: dry run**
```sql
COPY INTO emp_bad FROM @lab_stage/messy/
FILE_FORMAT = (FORMAT_NAME = 'csv_nh')
VALIDATION_MODE = RETURN_ERRORS;
```
Expected: errors pointing at lines 2, 3, and 4. Nothing is loaded.

**Step 3: load with the default behavior**
```sql
COPY INTO emp_bad FROM @lab_stage/messy/
FILE_FORMAT = (FORMAT_NAME = 'csv_nh');

SELECT COUNT(*) FROM emp_bad;
```
Expected: the load **fails**, and the count is **0**.

**Step 4: load with `ON_ERROR = CONTINUE`**
```sql
COPY INTO emp_bad FROM @lab_stage/messy/
FILE_FORMAT = (FORMAT_NAME = 'csv_nh')
ON_ERROR = CONTINUE;

SELECT * FROM emp_bad ORDER BY id;
```
Expected: status `PARTIALLY_LOADED`, 2 rows loaded, 3 errors seen. The table has Asha (1) and Divya (5).

**Step 5: find the bad rows with VALIDATE**
```sql
SELECT error, line, rejected_record
FROM TABLE(VALIDATE(emp_bad, JOB_ID => '_last'));
```
Expected: 3 rows, one per bad line, with `REJECTED_RECORD` showing the raw text. Three different error reasons: not a number, too many columns, too few columns.

**Step 6: quarantine them**
```sql
CREATE OR REPLACE TABLE emp_rejected AS
SELECT rejected_record, error, CURRENT_TIMESTAMP() AS captured_at
FROM TABLE(VALIDATE(emp_bad, JOB_ID => '_last'));

SELECT * FROM emp_rejected;
```

**Step 7: try the other ON_ERROR options** (reset between each, since a load is remembered in history):
```sql
TRUNCATE TABLE emp_bad;
COPY INTO emp_bad FROM @lab_stage/messy/
  FILE_FORMAT = (FORMAT_NAME = 'csv_nh') ON_ERROR = SKIP_FILE;
SELECT COUNT(*) FROM emp_bad;      -- expected: 0

TRUNCATE TABLE emp_bad;
COPY INTO emp_bad FROM @lab_stage/messy/
  FILE_FORMAT = (FORMAT_NAME = 'csv_nh') ON_ERROR = SKIP_FILE_4;
SELECT COUNT(*) FROM emp_bad;      -- expected: 2
```
If a COPY says "0 files processed" after a truncate, add `FORCE = TRUE` to that statement.

**Step 8: fix the rejected rows by hand**
```sql
INSERT INTO emp_bad VALUES (2,'Ravi','Chennai',62000),(3,'Meena','Hyderabad',58000),(4,'Kiran','Delhi',55000);
SELECT * FROM emp_bad ORDER BY id;     -- all 5 rows now
```

**Check yourself**
1. Why is the count 0 in Step 3? (The default `ABORT_STATEMENT` stops on the first bad row.)
2. Why does `SKIP_FILE_4` load 2 rows but `SKIP_FILE_2` loads none? (We have 3 bad rows: fewer than 4 but more than 2.)
3. Would `VALIDATE` work if the COPY had a `SELECT` transformation? (No.)
