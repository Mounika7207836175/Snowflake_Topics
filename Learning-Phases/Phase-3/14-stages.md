# Snowflake: Stages

## 1. What is a stage?

A stage provides access to files. Uploading a file does not automatically load its rows into a database table.

| Type | Where files live | What the stage does |
|---|---|---|
| Internal | Snowflake-managed storage | Holds uploaded files. |
| External | Azure Blob Storage in these examples | References a storage location and its access configuration. |

```text
Local file → Internal stage → COPY INTO → Table rows

Azure file → External stage → COPY INTO → Table rows
                           ↘ External table → Query external files
```

A file in an internal stage contains data, but it is still a file, not rows already loaded into a table. External tables require external stages; an internal stage is not a substitute.

## 2. Internal stage types

| Type | Reference | Purpose | Access |
|---|---|---|---|
| User stage | @~ | Personal file area automatically provided for my user. | Intended for that user. |
| Table stage | @%products | File area attached to the products table. | Requires table ownership. |
| Named stage | @products_stage | Separate stage created with CREATE STAGE. | Permissions can be granted separately to roles. |

- A user stage contains uploaded files, not information about the user.
- A table stage does not automatically collect files intended for its table; they must be uploaded.
- A named stage can supply files to multiple tables. The tables themselves are not inside the stage.
- A named stage is a schema-level object. User/table stages are implicit, not separately created named objects.
- Iceberg tables do not support table stages.

```text
@team_files
  products.csv  → PRODUCTS table
  customers.csv → CUSTOMERS table
```

COPY INTO chooses the destination table. Named stages are convenient for shared, repeatable pipelines.

```sql
-- List my user's staged files, not user-profile information.
LIST @~;

-- List files attached to the products table, not its stored rows.
-- Requires that the table exists and the active role owns it.
LIST @%products;

-- Create an independent internal stage that can serve multiple loads.
CREATE STAGE team_files;

-- Inspect this named stage's files.
LIST @team_files;
```

## 3. CSV fields: $1 versus VALUE:c1

| Query source | CSV field references |
|---|---|
| Direct stage query: FROM @stage_name | $1, $2, $3 |
| External table query: FROM external_table_name | VALUE:c1, VALUE:c2, VALUE:c3 |

This difference is about the object being queried, not internal versus external storage.

For `1,Keyboard,1500`, both examples below select Keyboard:

```sql
-- Read field 2 directly from a staged CSV file.
SELECT $2::VARCHAR AS product_name FROM @products_stage;

-- Equivalent access through an existing CSV external table.
-- Example only: products_external must be configured separately.
SELECT VALUE:c2::VARCHAR AS product_name FROM products_external;
```

For Parquet, the staged $1 represents a structured row, and fields are accessed by name, such as $1:countryOrRegion. File formats determine how content is interpreted.

## 4. Internal-stage example

Create a local file called products.csv:

```csv
product_id,product_name,price
1,Keyboard,1500
2,Mouse,500
```

Select an existing warehouse and use a role with the required privileges. Run object creation once; inspect existing objects rather than blindly replacing them.

```sql
-- Keep this exercise separate from earlier examples.
USE DATABASE LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS STAGE_LAB;
USE SCHEMA STAGE_LAB;

-- Skip the header and allow double-quoted CSV fields.
CREATE FILE FORMAT products_csv_format
  TYPE = CSV
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

-- Omitting URL creates an internal stage.
-- Associate parsing settings for direct staged-file queries.
CREATE STAGE products_stage
  FILE_FORMAT = (FORMAT_NAME = 'products_csv_format');
```

Open LEARNING_DB.STAGE_LAB.PRODUCTS_STAGE in Snowsight, select Upload Files, and upload products.csv. Uploading stores the file; the table has not been loaded yet.

```sql
-- Verify the file's presence and size without reading its records.
LIST @products_stage;

-- Inspect the fields before loading them into the destination.
-- Expected: two product rows without the CSV header.
SELECT
  $1::INT AS product_id,
  $2::VARCHAR AS product_name,
  $3::NUMBER(10,2) AS price
FROM @products_stage;

-- Define the destination columns and their data types.
CREATE TABLE products (
  product_id INT,
  product_name VARCHAR,
  price NUMBER(10,2)
);

-- Copy the specified file's rows into the table.
-- Abort this load if a data error occurs.
COPY INTO products
FROM @products_stage
FILES = ('products.csv')
FILE_FORMAT = (FORMAT_NAME = 'products_csv_format')
ON_ERROR = 'ABORT_STATEMENT';

-- Expected: two stored product rows.
SELECT * FROM products ORDER BY product_id;

-- The file remains because loading does not remove it by default.
LIST @products_stage;
```

## 5. Repeated loads and duplicate rows

The duplicate-file check happens when loading into a destination table, not when uploading files to a stage. Snowflake normally skips an unchanged file already loaded into that table while applicable load metadata is available. This is not a permanent row-level duplicate guarantee.

```sql
-- Retry the same unchanged file without forcing a reload.
-- Expected: no extra rows while its prior load is recognized.
COPY INTO products
FROM @products_stage
FILES = ('products.csv')
FILE_FORMAT = (FORMAT_NAME = 'products_csv_format');

-- Expected: still two rows if the initial load succeeded once.
SELECT COUNT(*) FROM products;
```

FORCE = TRUE overrides load-file protection and can create duplicates. Do not add it casually.

| Situation | Result to expect |
|---|---|
| Same unchanged file loaded into the same table | Normally skipped while prior-load metadata applies. |
| Same rows under a new filename | Can be loaded again, producing duplicate rows. |
| FORCE = TRUE | Can reload previously loaded files. |

Example: products.csv contains Keyboard and Mouse; products_again.csv contains the same two rows. Loading both can produce four rows. Use source keys, deduplication, and suitable merge logic when row-level repeat protection is required.

## 6. File cleanup versus table data

```sql
-- Verify the load before deleting the practice file.
SELECT * FROM products ORDER BY product_id;

-- Delete only the CSV used in this lab from the internal stage.
REMOVE @products_stage PATTERN = '^products[.]csv$';

-- Expected: the CSV is absent from the stage.
LIST @products_stage;

-- Expected: the two already-loaded rows remain in the table.
SELECT * FROM products ORDER BY product_id;
```

Once loaded, staged files and table rows are separate copies. Removing a staged file does not delete loaded rows. In contrast, an external table reads source files: deleting an Azure source file affects the data available through that external table.

## 7. External-stage example from the earlier topic

This example uses public Azure files and therefore does not configure private storage credentials. For private Azure storage, configure storage access and permissions separately.

```sql
-- Use the existing external-table exercise schema.
USE DATABASE LEARNING_DB;
USE SCHEMA EXTERNAL_TABLE_LAB;

-- Describe the public dataset's file format.
CREATE FILE FORMAT IF NOT EXISTS holidays_parquet_format TYPE = PARQUET;

-- URL makes this a reference to Azure storage rather than an internal upload area.
CREATE STAGE IF NOT EXISTS azure_holidays_stage
  URL = 'azure://azureopendatastorage.blob.core.windows.net/holidaydatacontainer/Processed/'
  FILE_FORMAT = (FORMAT_NAME = 'holidays_parquet_format');

-- Check the accessible files before trying to parse them.
LIST @azure_holidays_stage;

-- Select only Parquet-named files to avoid nonmatching empty marker files.
-- PATTERN filters names, not file sizes or content validity.
SELECT METADATA$FILENAME AS source_file, $1 AS file_row
FROM @azure_holidays_stage (PATTERN => '.*[.]parquet')
LIMIT 5;
```

METADATA$FILENAME identifies the file supplying a row; it can repeat across many rows. An empty file still has a name but cannot be parsed as a valid Parquet file. A zero-byte file ending in .parquet would still match this pattern.

## 8. Commands to recognize

| Command | Purpose |
|---|---|
| CREATE STAGE | Create a named stage. |
| LIST @stage | List files. |
| PUT | Upload local files to an internal stage through a supported client. |
| GET | Download internal-stage files through a supported client. |
| SELECT ... FROM @stage | Inspect supported structured file contents. |
| COPY INTO table | Load file data into table rows. |
| COPY INTO @stage | Export query results into files. |
| REMOVE @stage | Delete staged files; use a narrow path/pattern. |

Snowsight provides file upload controls. A local-file PUT is not run from a Snowsight SQL worksheet. Azure uploads use Azure tools rather than PUT to an external stage.

## 9. Optional follow-up

- Private Azure storage integrations and permissions.
- PUT/GET with a supported command-line client.
- Exporting query results with COPY INTO @stage.
- Detailed load-history retention and recovery behavior.

## Quick recall

Stage = files. Table = loaded rows. File format = reading instructions. COPY INTO = loading or exporting. External table = querying external files through a table definition.

## References

- [Internal stage choices](https://docs.snowflake.com/en/user-guide/data-load-local-file-system-create-stage)
- [Upload files using Snowsight](https://docs.snowflake.com/en/user-guide/data-load-local-file-system-stage-ui)
- [CREATE STAGE](https://docs.snowflake.com/en/sql-reference/sql/create-stage)
- [COPY INTO table](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)

Reviewed: 7 October 2026.
