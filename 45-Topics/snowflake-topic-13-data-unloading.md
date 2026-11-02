# Snowflake Notes: Topic 13, Data Unloading

**Unloading** is the reverse of loading: you take data **out of Snowflake** and write it as files. Loading was `COPY INTO table`. Unloading is **`COPY INTO location`**.

## 1. COPY INTO location

```sql
COPY INTO @lab_stage/unload/orders/
FROM orders_big                      -- a table, or a query in brackets: FROM (SELECT ...)
FILE_FORMAT = (TYPE = CSV)
HEADER = TRUE;
```
- **Where the files go:** an internal stage, an external stage, or straight to cloud storage (section 2).
- **What goes in:** a whole table, or the result of any query. A query lets you pick columns, filter, and join before exporting.
- The result shows `rows_unloaded`, `input_bytes`, and `output_bytes`.
- File names look like `data_<query-id>_0_0_0.csv.gz`. The query id makes every unload produce unique names.

**Real scenarios:**
- Nightly export of yesterday's orders to a data lake, for Spark or Databricks to read.
- A monthly report as one CSV file for a business partner.
- Archiving old data before deleting it.

## 2. Unloading to Azure (and S3 or GCS)

Same command, different location.

**Azure:**
```sql
-- Option 1: through a stage you already created
COPY INTO @azure_out/orders/ FROM orders_big FILE_FORMAT = (TYPE = PARQUET);

-- Option 2: straight to the container, using a storage integration
COPY INTO 'azure://mystorageacct.blob.core.windows.net/snowflake-data/orders/'
FROM orders_big
STORAGE_INTEGRATION = azure_int
FILE_FORMAT = (TYPE = PARQUET);
```
**S3:** `COPY INTO 's3://my-bucket/orders/' ... STORAGE_INTEGRATION = s3_int`
**GCS:** `COPY INTO 'gcs://my-bucket/orders/' ... STORAGE_INTEGRATION = gcs_int`

**Permission gotcha (very common):** when you **loaded** from Azure, read access was enough (Storage Blob Data **Reader**). To **unload**, Snowflake must **write**, so the Azure role must be **Storage Blob Data Contributor**, or the SAS token must include **Write** and **Create** permissions. If you only gave Reader, the unload fails with a permission error.

After unloading, open **Azure Storage Explorer** and look in the container. You'll see the files (and folders, if you used PARTITION BY).

To bring files from an internal stage down to your laptop: the `GET` command in SnowSQL, for example `GET @lab_stage/unload/orders/ file://C:\downloads\`.

## 3. Single file vs multiple files

By default, Snowflake writes **many files in parallel** (each server thread in the warehouse writes its own). That's fast. Sometimes you need exactly one file.

| | Multiple files (default) | Single file (`SINGLE = TRUE`) |
|---|---|---|
| Speed | Fast, parallel | Slower, one writer |
| File name | Generated (`data_..._0_0_0`) | The name **you** give in the path |
| Best for | Data lakes, Spark, big data | Reports for people or partners |

```sql
COPY INTO @lab_stage/unload/single/orders_report.csv
FROM (SELECT * FROM orders_big WHERE order_date < '2023-01-11')
FILE_FORMAT = (TYPE = CSV COMPRESSION = NONE)
HEADER = TRUE
SINGLE = TRUE;
```
The file is called exactly `orders_report.csv`.

**Gotcha:** a single file can't be bigger than `MAX_FILE_SIZE` (next section). If your data is larger, raise it or the unload fails.

## 4. MAX_FILE_SIZE

The **upper size limit of each file**, in bytes.
- Default: **16 MB** (16777216).
- Maximum: 5 GB on S3, GCS, and Azure.
- A smaller value gives **more, smaller files**. A larger value gives **fewer, bigger files**.

```sql
MAX_FILE_SIZE = 134217728      -- 128 MB
```

**How to choose:**
- For data lake files read by Spark or similar tools, aim for **roughly 100 to 250 MB** per file. Thousands of tiny files slow everything down, and so does one giant file.
- With `SINGLE = TRUE`, set it large enough for the whole file.
- The actual number of files also depends on the warehouse size, because bigger warehouses write with more parallel threads.

## 5. PARTITION BY

Splits the output into **folders**, based on a value from each row, so the files are organized like a lake.

```sql
PARTITION BY ('year=' || TO_VARCHAR(order_date, 'YYYY') || '/month=' || TO_VARCHAR(order_date, 'MM'))
```
This produces folders like:
```
orders_parquet/
 ├── year=2023/month=01/data_..._0_0_0.snappy.parquet
 ├── year=2023/month=02/data_..._0_0_0.snappy.parquet
 └── year=2024/month=12/data_..._0_0_0.snappy.parquet
```
**Why companies do this:** engines like Spark, Athena, and Snowflake external tables can **skip** whole folders. A query asking for March 2024 reads only the `year=2024/month=03/` folder, not the whole dataset. This is called **partition pruning**.

Rules and tips:
- Use the `name=value` folder style (called Hive-style), because most tools understand it.
- Don't combine `PARTITION BY` with `SINGLE = TRUE`.
- Rows where the partition expression is NULL go into a special folder, so check your `LIST` output.
- **Don't over-partition.** Partitioning by day and customer id could create millions of tiny files. Choose partitions with a decent amount of data each (usually date, sometimes region).

## 6. File formats and compression

Unloading supports only **CSV, JSON, and Parquet**. (Avro and ORC are not supported for unloading.)

| Format | Good for | Notes |
|---|---|---|
| **CSV** | Reports, people, simple exchange | Use `HEADER = TRUE` for column names |
| **JSON** | APIs, nested data | Query must return one VARIANT/OBJECT column |
| **Parquet** | Data lakes, Spark, Databricks | Columnar, compressed. **Use `HEADER = TRUE`** so the column names are kept (otherwise you get generic names like `_COL_0`) |

**Compression:**

| Format | Default | Other options |
|---|---|---|
| CSV, JSON | `GZIP` | `BZ2`, `ZSTD`, `DEFLATE`, `RAW_DEFLATE`, `NONE` |
| Parquet | `SNAPPY` | Others available, SNAPPY is the usual choice |

Trade-off in plain words:
- **None:** biggest files, nothing to decompress.
- **GZIP:** small files, a bit slower, works everywhere. Typical for CSV.
- **ZSTD:** small files and fast. A good modern choice.
- **SNAPPY:** moderately small and very fast. The standard for Parquet.

Smaller files mean lower storage and transfer cost, but the receiving tool must be able to read that compression.

**Useful extra options**
| Option | What it does |
|---|---|
| `OVERWRITE = TRUE` | Replace existing files with the same name |
| `DETAILED_OUTPUT = TRUE` | Returns one row **per file** (name, size, row count), not just totals |
| `INCLUDE_QUERY_ID` | Controls whether the query id appears in file names |

---

## Quick recap
1. `COPY INTO location` exports a table or query result to files in a stage or cloud storage (Azure, S3, GCS).
2. Unloading to Azure needs **write** permission (Contributor role, or SAS with Write and Create).
3. Default is many parallel files. `SINGLE = TRUE` gives one named file (limited by `MAX_FILE_SIZE`).
4. `MAX_FILE_SIZE`: default 16 MB, up to 5 GB. For lakes aim for about 100 to 250 MB per file.
5. `PARTITION BY` writes `name=value` folders, so tools can skip data they don't need.
6. Unload formats: CSV, JSON, Parquet. Use `HEADER = TRUE` for Parquet. Defaults: GZIP for CSV and JSON, SNAPPY for Parquet.

---

## Hands-on: unload a table to partitioned Parquet files

**Step 0: setup**
```sql
USE WAREHOUSE learn_wh;
CREATE DATABASE IF NOT EXISTS practice_db;
CREATE SCHEMA IF NOT EXISTS practice_db.lab;
USE SCHEMA practice_db.lab;
CREATE STAGE IF NOT EXISTS lab_stage;
CREATE OR REPLACE FILE FORMAT parquet_ff TYPE = PARQUET;
```

**Step 1: create a table with 3000 orders over two years**
```sql
CREATE OR REPLACE TABLE orders_big AS
WITH g AS (
  SELECT ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1 AS n
  FROM TABLE(GENERATOR(ROWCOUNT => 3000))
)
SELECT n + 1 AS order_id,
       DATEADD(day, MOD(n, 730), '2023-01-01'::DATE) AS order_date,
       CASE MOD(n, 3) WHEN 0 THEN 'East' WHEN 1 THEN 'West' ELSE 'North' END AS region,
       MOD(n * 7, 500) + 10 AS amount
FROM g;

SELECT COUNT(*) AS total_rows, YEAR(order_date) AS yr
FROM orders_big GROUP BY YEAR(order_date) ORDER BY yr;
```
Expected: 2023 has **1540** rows, 2024 has **1460** rows (3000 in total).

**Step 2: single file (a report)**
```sql
COPY INTO @lab_stage/unload/single/orders_report.csv
FROM (SELECT * FROM orders_big WHERE order_date < '2023-01-11')
FILE_FORMAT = (TYPE = CSV COMPRESSION = NONE)
HEADER = TRUE
SINGLE = TRUE
OVERWRITE = TRUE;

LIST @lab_stage/unload/single/;
```
Expected: **50 rows unloaded**, and the listing shows **one** file named `orders_report.csv`.

**Step 3: multiple files and MAX_FILE_SIZE**
```sql
COPY INTO @lab_stage/unload/multi/
FROM orders_big
FILE_FORMAT = (TYPE = CSV COMPRESSION = NONE)
HEADER = TRUE
MAX_FILE_SIZE = 10000
DETAILED_OUTPUT = TRUE;

LIST @lab_stage/unload/multi/;
```
Expected: **more than one file** (the exact count varies). The detailed output shows rows per file, and the row counts add up to 3000. Try again with `MAX_FILE_SIZE = 1000000` and compare the number of files.

**Step 4: compare compression**
```sql
COPY INTO @lab_stage/unload/c_none/ FROM orders_big
  FILE_FORMAT = (TYPE = CSV COMPRESSION = NONE) HEADER = TRUE SINGLE = TRUE OVERWRITE = TRUE;
COPY INTO @lab_stage/unload/c_gzip/ FROM orders_big
  FILE_FORMAT = (TYPE = CSV COMPRESSION = GZIP) HEADER = TRUE SINGLE = TRUE OVERWRITE = TRUE;
COPY INTO @lab_stage/unload/c_zstd/ FROM orders_big
  FILE_FORMAT = (TYPE = CSV COMPRESSION = ZSTD) HEADER = TRUE SINGLE = TRUE OVERWRITE = TRUE;
COPY INTO @lab_stage/unload/c_parquet/ FROM orders_big
  FILE_FORMAT = (TYPE = PARQUET) HEADER = TRUE SINGLE = TRUE OVERWRITE = TRUE;

LIST @lab_stage/unload/ PATTERN = '.*c_.*';
```
Expected: the **uncompressed CSV is the largest**, and the gzip, zstd, and Parquet files are much smaller. Note the sizes in a small table.

**Step 5 (the main task): partitioned Parquet**
```sql
COPY INTO @lab_stage/unload/orders_parquet/
FROM (SELECT * FROM orders_big)
PARTITION BY ('year=' || TO_VARCHAR(order_date, 'YYYY') || '/month=' || TO_VARCHAR(order_date, 'MM'))
FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY)
HEADER = TRUE
MAX_FILE_SIZE = 134217728
OVERWRITE = TRUE;
```
Expected: **3000 rows unloaded**.

**Step 6: look at the folders**
```sql
LIST @lab_stage/unload/orders_parquet/;

SELECT COUNT(*) AS files,
       COUNT(DISTINCT REGEXP_SUBSTR("name", 'year=[0-9]+/month=[0-9]+')) AS partitions
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```
Expected: **24 partitions** (January 2023 to December 2024), and at least 24 files.

**Step 7: read the data back, and see partition pruning**
```sql
-- all rows
SELECT COUNT(*) FROM @lab_stage/unload/orders_parquet/ (FILE_FORMAT => 'parquet_ff');

-- only one month, using the folder name
SELECT COUNT(*)
FROM @lab_stage/unload/orders_parquet/ (FILE_FORMAT => 'parquet_ff')
WHERE METADATA$FILENAME ILIKE '%year=2024/month=03/%';

SELECT COUNT(*)
FROM @lab_stage/unload/orders_parquet/ (FILE_FORMAT => 'parquet_ff')
WHERE METADATA$FILENAME ILIKE '%year=2023/month=01/%';
```
Expected: **3000**, then **124** (March 2024), then **155** (January 2023).

**Step 8: load it back and verify nothing was lost**
```sql
CREATE OR REPLACE TABLE orders_check LIKE orders_big;

COPY INTO orders_check
FROM @lab_stage/unload/orders_parquet/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT (SELECT COUNT(*) FROM orders_big)   AS original_rows,
       (SELECT COUNT(*) FROM orders_check) AS reloaded_rows,
       (SELECT SUM(amount) FROM orders_big)   -
       (SELECT SUM(amount) FROM orders_check) AS amount_difference;
```
Expected: both counts **3000**, and the difference **0**.

**Optional (needs an Azure account): unload to Azure**
Create a stage pointing at a container with **write** access (storage integration with the Contributor role, or a SAS token with Write and Create), then:
```sql
COPY INTO @azure_out/orders/
FROM orders_big
PARTITION BY ('year=' || TO_VARCHAR(order_date, 'YYYY') || '/month=' || TO_VARCHAR(order_date, 'MM'))
FILE_FORMAT = (TYPE = PARQUET)
HEADER = TRUE;
```
Open Azure Storage Explorer and refresh the container to see the `year=.../month=...` folders.

**Optional: download to your laptop (SnowSQL)**
```sql
GET @lab_stage/unload/single/orders_report.csv file://C:\downloads\;
```

**Cleanup:** `DROP TABLE orders_big; DROP TABLE orders_check;` and `REMOVE @lab_stage/unload/;`

**Check yourself**
1. Why do you need `HEADER = TRUE` for Parquet? (Without it the column names are lost and replaced by generic names.)
2. Why did Step 5 create 24 folders? (There are 24 different year and month values.)
3. When would you use `SINGLE = TRUE`, and what's the catch? (For one named report file. The file can't exceed `MAX_FILE_SIZE`, and it is slower.)
4. Your Azure unload fails with a permission error even though loading worked. What's the likely cause? (The role or SAS token has only read access, and unloading needs write.)
