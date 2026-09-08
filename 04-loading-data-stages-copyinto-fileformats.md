# Topic 4: Loading Data — Stages, COPY INTO, and File Formats

## Chunk 1: What is a Stage, and why do we need it?

Data starts as **files** (CSV, JSON, etc.) sitting outside Snowflake — on a laptop or in cloud storage. Before that data can go into a Snowflake **table**, it has to sit somewhere temporarily so Snowflake can read and pull it in. That temporary waiting area is called a **Stage**.

**Analogy:** A stage is a **loading dock** outside a warehouse. A truck (file) pulls up to the dock first. Workers then unload its contents onto the warehouse shelves (the table).

Flow: **File (outside Snowflake) → Stage (drop-off point) → Table (final storage)**

---

## Chunk 2: The Two Types of Stages

**1. Internal Stage** — Snowflake manages this storage for you, inside Snowflake itself. No separate cloud account needed.

**2. External Stage** — points to storage you already own (AWS S3, Azure Blob, GCS). Snowflake doesn't store the file — it just knows where to look and reads from your bucket directly.

**Analogy:**
- Internal Stage = dropping your package at the warehouse's own front desk.
- External Stage = the warehouse partners with a nearby locker facility (that you rent) and staff grab your package from there.

---

## Chunk 3: The Three Kinds of Internal Stage

**1. User Stage** — automatically assigned to every user when they join Snowflake. Private, like a personal locker / ID card.

**2. Table Stage** — automatically created for every table. Tied to that one specific table, like a locker glued to one shelf.

**3. Named Stage** — the only type YOU create explicitly (`CREATE STAGE`). Reusable across users/tables, like a shared locker room.

**Key point:** User and Table stages already exist automatically. A Named Stage is the only one you build yourself.

---

## Chunk 4: Storage Integration — The Secure Way to Connect to External Storage

An External Stage needs permission to access your cloud storage.

**Risky way:** Typing cloud storage access keys directly into SQL. If that script is shared or pushed to GitHub, your storage credentials are exposed.

**Professional way — Storage Integration:** A Snowflake object that securely links your Snowflake account to your cloud account via a trust relationship (e.g., an AWS IAM role) — a verified handshake. No password/key is ever stored in the object or exposed in SQL code.

**Note — Staging vs Storage:**
- **Storage** = the general place where data/files are kept (Snowflake's own storage, or your S3 bucket, etc.)
- **Staging** = the specific *purpose*: a temporary drop-off point used for loading/unloading data. A Stage object *points to* a storage location for this purpose.

---

## Chunk 5: File Formats — Telling Snowflake How to Read a File

Files differ in structure (delimiters, headers, quoting, compression, file type — CSV/JSON/**Parquet**, a compressed column-based format optimized for big data). A **File Format** object defines these reading rules once, so you don't redefine them on every load.

**Analogy:** A recipe card you write once and reuse for every cook, instead of re-explaining instructions each time.

Rules a File Format can define: delimiter character, header row skip, quoted text handling, compression, file type (CSV/JSON/Parquet/etc.).

---

## Chunk 6: PUT — Getting a File From Your Laptop Into a Stage

If a file is on your own laptop, use **`PUT`** (run via SnowSQL, Snowflake's command-line client — not regular SQL) to upload it into an **Internal Stage** first.

```
PUT file:///C:/Users/Mounika/Desktop/orders.csv @my_internal_stage;
```

`PUT` only **places** the file into the stage — it does NOT load it into a table. That's a separate step.

**Skip this step entirely** if using an External Stage — the file is already in the cloud; Snowflake reads it directly.

Full flow for a laptop file: **Laptop file → `PUT` → Internal Stage → `COPY INTO` → Table**

---

## Chunk 7: COPY INTO — What It Does (Concept)

`COPY INTO` is the command that physically moves data **from the stage into the table** — the moment workers carry boxes from the dock onto the warehouse shelves.

Key behaviors:
- **Load history tracking**: Snowflake remembers which files were already loaded (via file metadata). Re-running the same `COPY INTO` won't duplicate data — it automatically skips already-loaded files. This prevents accidental double-counting if a job runs twice.
- **Flexible error handling**: choose whether to stop the whole load, skip just bad rows, or skip the whole file on error.
- **Dry-run capability**: test what would fail, without actually loading anything.

---

## Chunk 8: Putting It All Together — The Code

**Step 1: Create a Named Internal Stage**
```sql
CREATE STAGE my_internal_stage;
```

**Step 2: Create an External Stage connected to S3, the secure way**

Storage Integration (the secure handshake object):
```sql
CREATE STORAGE INTEGRATION s3_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/snowflake-role'
  ENABLED = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('s3://my-company-bucket/raw-data/');
```
- `STORAGE_AWS_ROLE_ARN` — the verified identity on the AWS side that Snowflake is allowed to assume/trust (provided by your AWS admin).
- `STORAGE_ALLOWED_LOCATIONS` — restricts the integration to a specific bucket/folder only (least privilege security practice).

External Stage using that integration:
```sql
CREATE STAGE my_external_stage
  URL = 's3://my-company-bucket/raw-data/'
  STORAGE_INTEGRATION = s3_int;
```

**Step 3: Create a File Format**
```sql
CREATE FILE FORMAT csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', 'null', '');
```
- `SKIP_HEADER = 1` — ignore the first row (column names).
- `FIELD_OPTIONALLY_ENCLOSED_BY = '"'` — handles quoted text values, e.g. `"Smith, John"`, so the internal comma isn't mistaken for a delimiter.
- `NULL_IF` — which text values should be treated as actual NULL instead of literal text.

**Step 4: PUT a local file into the internal stage** (only if file starts on your laptop)
```
PUT file:///C:/Users/Mounika/Desktop/orders.csv @my_internal_stage;
```

---

## Chunk 9: COPY INTO — The Syntax

```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_external_stage
FILE_FORMAT = (FORMAT_NAME = csv_format)
ON_ERROR = 'CONTINUE';
```
- `COPY INTO SALES_DB.RAW.ORDERS_RAW` — destination table (full `DATABASE.SCHEMA.TABLE` address).
- `FROM @my_external_stage` — source stage (`@` always denotes a stage).
- `FILE_FORMAT = (FORMAT_NAME = csv_format)` — which reading rules to apply.
- `ON_ERROR = 'CONTINUE'` — skip broken rows instead of stopping the whole load.

**Loading only specific files with `PATTERN`:**
```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_external_stage
FILE_FORMAT = (FORMAT_NAME = csv_format)
PATTERN = '.*orders_2026.*\.csv'
ON_ERROR = 'CONTINUE';
```

**Dry-run before committing, using `VALIDATION_MODE`:**
```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_external_stage
FILE_FORMAT = (FORMAT_NAME = csv_format)
VALIDATION_MODE = 'RETURN_ERRORS';
```
Shows what would fail without loading anything.

**General shape to remember (don't memorize exact syntax):**
`COPY INTO <table> FROM <stage> FILE_FORMAT = (...) <options>`

---

## Glossary — New Terms Introduced
| Term | Meaning |
|---|---|
| Stage | Temporary waiting area for files before/after loading into/from tables |
| Internal Stage | Stage storage managed by Snowflake itself |
| External Stage | Stage pointing to storage you own (S3/Blob/GCS) |
| User Stage | Auto-created personal stage per user |
| Table Stage | Auto-created stage tied to one specific table |
| Named Stage | Manually created, reusable stage |
| Storage Integration | Secure trust link between Snowflake and a cloud account (no exposed credentials) |
| IAM Role | AWS identity/permission construct used to grant trusted access |
| File Format | Object defining rules for parsing a file (delimiter, header, quoting, compression, type) |
| Parquet | A compressed, column-based file format optimized for large datasets |
| PUT | Command (via SnowSQL) to upload a local file into an internal stage |
| COPY INTO | Command that loads data from a stage into a table |
| Load history tracking | Snowflake's automatic tracking of already-loaded files, preventing duplicate loads |
| PATTERN | Regex filter to control which files in a stage get loaded |
| ON_ERROR | Controls error handling during load: ABORT_STATEMENT / CONTINUE / SKIP_FILE |
| VALIDATION_MODE | Dry-run mode that reports errors without actually loading data |

## Learning Note
Don't memorize exact syntax/keywords — even experienced engineers look these up every time. Focus on remembering: what each object/command *does* and *why* it exists. The rough shape of commands becomes familiar through repeated practice over time.

---
**Next Topic:** Semi-Structured Data — VARIANT type, JSON/Parquet handling
