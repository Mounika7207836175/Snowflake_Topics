# Snowflake Notes: Topic 9, Loading Data into Snowflake

## The big picture: 3 steps

**Think of a warehouse delivery.** Trucks unload boxes at a **loading dock** (the stage). A worker follows **unpacking instructions** (the file format). Then the goods go onto the **shelf** (the table).

1. **Get the files into a stage** (internal or external)
2. **Tell Snowflake how to read them** (file format)
3. **Run `COPY INTO table`** to load them

---

## Part 1: Internal stage, a full load from start to finish

**Step 1: create the target table**
```sql
CREATE OR REPLACE TABLE emp (id INT, name STRING, city STRING, salary NUMBER);
```

**Step 2: file format and stage**
```sql
CREATE OR REPLACE FILE FORMAT csv_ff
  TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE OR REPLACE STAGE my_stage FILE_FORMAT = csv_ff;
```

**Step 3: upload from your laptop (SnowSQL)**
```sql
PUT file://C:\snowflake_lab\employees.csv @my_stage;
```

**Step 4: load**
```sql
COPY INTO emp FROM @my_stage;
```
The result shows one row per file:

| file | status | rows_parsed | rows_loaded | errors_seen |
|---|---|---|---|---|
| employees.csv.gz | LOADED | 3 | 3 | 0 |

**Step 5: check**
```sql
SELECT * FROM emp;
```

### What happens if you run COPY again?
Result: **"Copy executed with 0 files processed."**

Snowflake remembers which files it already loaded (for 64 days) and **skips them**, so you don't get duplicates. To force a reload (which creates duplicates), add `FORCE = TRUE`.

### Useful COPY options
| Option | What it does |
|---|---|
| `ON_ERROR = ABORT_STATEMENT` | **Default.** Stop everything if any row is bad |
| `ON_ERROR = CONTINUE` | Skip bad rows, load the rest |
| `ON_ERROR = SKIP_FILE` | Skip any file that has a bad row |
| `PURGE = TRUE` | Delete the file from the stage after a successful load |
| `FORCE = TRUE` | Load files again, even if loaded before |
| `FILES = ('a.csv.gz')` | Load only specific files |
| `PATTERN = '.*sales.*[.]csv[.]gz'` | Load only files whose names match |
| `VALIDATION_MODE = RETURN_ERRORS` | **Dry run:** check for errors without loading anything |

### Example: a bad row
Suppose the file has `4,Kiran,Delhi,abc` (salary is text).
- Default: the load **fails** with an error like "Numeric value 'abc' is not recognized".
- With `ON_ERROR = CONTINUE`: the bad row is skipped, the other rows load.

To see which rows were rejected in the last load:
```sql
SELECT * FROM TABLE(VALIDATE(emp, JOB_ID => '_last'));
```

**Scenario:** a daily sales file arrives. Run a dry run first:
```sql
COPY INTO emp FROM @my_stage VALIDATION_MODE = RETURN_ERRORS;
```
If it shows no errors, run the real COPY.

---

## Part 2: External stage with Azure

**Why:** in most real projects, files already sit in the company's cloud storage (Azure Blob / ADLS), dropped by other systems. You don't use PUT. You point Snowflake at that location.

| | Internal stage | External stage |
|---|---|---|
| Where files live | Inside Snowflake storage | In **your** Azure storage (or S3, GCS) |
| How files get there | `PUT` from your laptop | Upload to Azure, or another system writes them |
| Who manages files | Snowflake | You (in Azure) |
| Storage billed by | Snowflake | Azure |
| Typical use | Manual and small loads | Real pipelines, data lakes |

### What is Azure Storage Explorer?
A free **Microsoft desktop app** (not part of Snowflake) that lets you browse and manage Azure storage like a file manager. You use it to **upload files to a container**.

**How the Azure side is organized:**
```
Storage account  (mystorageacct)
 └── Container   (snowflake-data)      <- like a top-level folder
      └── folder (employees/)
           └── employees.csv
```

**Uploading with Storage Explorer:**
1. Open it and sign in with your Azure account.
2. Expand **Storage Accounts**, then your account, then **Blob Containers**.
3. Open (or create) your container, for example `snowflake-data`.
4. Click **Upload**, then **Upload Files**, and pick your file.

### Connecting Snowflake to that container

**Option 1: SAS token (quick, good for learning)**
A **SAS token** is a temporary password-like string, created in Azure, that grants limited access to the container.
```sql
CREATE OR REPLACE STAGE azure_stage
  URL = 'azure://mystorageacct.blob.core.windows.net/snowflake-data/employees/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<your-sas-token>')
  FILE_FORMAT = csv_ff;
```
Downsides: the token is typed into SQL (visible to anyone who can see the stage definition), and it expires.

**Option 2: Storage integration (recommended in real projects)**
No secrets in SQL. Two teams cooperate, which is very common in jobs:

*Snowflake side (needs the ACCOUNTADMIN role):*
```sql
CREATE STORAGE INTEGRATION azure_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = '<your-tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://mystorageacct.blob.core.windows.net/snowflake-data/');

DESC STORAGE INTEGRATION azure_int;
```
`DESC` shows two values: **AZURE_CONSENT_URL** and **AZURE_MULTI_TENANT_APP_NAME**.

*Azure side (an Azure admin does this):*
1. Open the **consent URL** in a browser and approve.
2. In the storage account, go to **Access control (IAM)**, then add a **role assignment**: give the Snowflake app (the name shown above) the role **Storage Blob Data Reader** (use *Contributor* if Snowflake must also write files).

*Back in Snowflake:*
```sql
CREATE OR REPLACE STAGE azure_stage
  URL = 'azure://mystorageacct.blob.core.windows.net/snowflake-data/employees/'
  STORAGE_INTEGRATION = azure_int
  FILE_FORMAT = csv_ff;
```

### Load from the external stage
```sql
LIST @azure_stage;                 -- see the files Snowflake can reach
COPY INTO emp FROM @azure_stage;   -- same COPY command as before
```
Everything after the stage is identical to the internal stage flow.

If `LIST` shows nothing or errors with "access denied", the usual causes are: wrong URL, an expired SAS token, or the Azure role not yet assigned.

---

## Part 3: Avro, Parquet, ORC

**CSV and JSON are text files.** Avro, Parquet, and ORC are **binary** files that carry their **own schema** (column names and types are stored inside the file). Snowflake can read the structure automatically.

| Format | Layout | Comes from | Typical use |
|---|---|---|---|
| **Avro** | Row-based, schema in the file header | Kafka and streaming systems | Streaming events, records written one at a time |
| **Parquet** | Columnar, compressed | Spark, Databricks, data lakes | **Most common** in data engineering, great for analytics |
| **ORC** | Columnar, compressed | Hadoop and Hive world | Older Hive-based pipelines |

*Why companies love Parquet:* smaller files (compressed), fast to read, the schema comes with the file, and no delimiter or quote problems like CSV.

### Three ways to load Parquet (Avro and ORC work the same, only the TYPE changes)

**Way 1: load everything into one VARIANT column**
```sql
CREATE OR REPLACE TABLE raw_events (v VARIANT);

COPY INTO raw_events
FROM @azure_stage/events/
FILE_FORMAT = (TYPE = PARQUET);

SELECT v:id::INT AS id, v:name::STRING AS name FROM raw_events;
```
Each file row becomes one VARIANT object. Same `col:key::TYPE` syntax from Topic 5.

**Way 2: match file columns to table columns by name**
```sql
COPY INTO emp
FROM @azure_stage/events/
FILE_FORMAT = (TYPE = PARQUET)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```
The table `emp` has real columns, and Snowflake matches each to the column with the same name inside the Parquet file.

**Way 3: pick and convert columns during the load**
```sql
COPY INTO emp (id, name, salary)
FROM (
  SELECT $1:id::INT, $1:name::STRING, $1:salary::NUMBER
  FROM @azure_stage/events/
)
FILE_FORMAT = (TYPE = PARQUET);
```
Use this when the table needs only some columns, or you must cast types.

### INFER_SCHEMA: let Snowflake read the columns for you
```sql
CREATE OR REPLACE FILE FORMAT parquet_ff TYPE = PARQUET;

SELECT *
FROM TABLE(INFER_SCHEMA(
  LOCATION => '@azure_stage/events/',
  FILE_FORMAT => 'parquet_ff'
));
```
The result lists each column name, its type, and whether it can be null. Then create the table automatically:
```sql
CREATE OR REPLACE TABLE emp_auto USING TEMPLATE (
  SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
  FROM TABLE(INFER_SCHEMA(
    LOCATION => '@azure_stage/events/',
    FILE_FORMAT => 'parquet_ff'
  ))
);
```
Very handy when a file has 80 columns and you don't want to type them.

### File size tip
Aim for files of roughly **100 to 250 MB compressed**. Thousands of tiny files load slowly, and one giant file can't be loaded in parallel. Snowflake loads multiple files at the same time, so a few well-sized files are best.

---

## Quick recap
1. Loading = files into a **stage**, a **file format** to read them, then `COPY INTO`.
2. Internal stage: `PUT` from your laptop. External stage: files already live in Azure/S3/GCS.
3. Snowflake remembers loaded files for 64 days and skips them. `FORCE` reloads them (duplicates).
4. `ON_ERROR`, `VALIDATION_MODE`, and `VALIDATE()` control and inspect bad rows.
5. Azure Storage Explorer is a Microsoft tool used to upload files into a container. It is not Snowflake.
6. External stage access: SAS token (quick) or storage integration (recommended, needs an Azure admin to approve).
7. Avro, Parquet, ORC are binary and carry their own schema. Load into VARIANT, with `MATCH_BY_COLUMN_NAME`, or with a SELECT inside COPY. `INFER_SCHEMA` reads the columns for you.

---

## Hands-on options (do later on a laptop)
1. **Internal stage load:** load `employees.csv`, run COPY twice to see the "0 files processed" skip, then try `FORCE = TRUE` and count the duplicates.
2. **Bad row test:** add a row with text in the salary column, run with `VALIDATION_MODE`, then with `ON_ERROR = CONTINUE`, and read the rejected rows with `VALIDATE`.
3. **Azure external stage:** upload a file with Azure Storage Explorer (a free Azure account is enough), create an external stage with a SAS token, and load it.
4. **Parquet:** use `INFER_SCHEMA` on a Parquet file, create the table with `USING TEMPLATE`, and load it.
