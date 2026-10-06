# Snowflake external tables

## Core idea
An external table exposes files in Azure Blob Storage as SQL rows without loading those rows into a standard Snowflake table. Snowflake stores file metadata and table definitions; the source files remain in Azure. External tables are read-only: INSERT, UPDATE, and DELETE cannot modify their source data.

Azure Blob Storage can underpin a data lake—a repository of data used for processing and analysis. An upstream application or pipeline is a system earlier in the data flow that produces these files.

## Objects and responsibilities
| Object | Responsibility |
|---|---|
| Storage integration | Configures secure cloud access; Azure must also authorize the identity Snowflake uses. |
| External stage | Identifies the external storage location and references access configuration. |
| File format | Defines how to interpret files, such as CSV delimiter or Parquet format. |
| External table | Exposes file contents through SQL and tracks file metadata. |

Public storage can allow access without a storage integration. Successful stage creation alone does not prove file access: verify with LIST.

## Fields, types, and file formats
- CSV is plain text. FIELD_DELIMITER = '|' separates fields using a pipe; SKIP_HEADER = 1 skips the first line but does not create column names.
- For staged CSV queries, $1, $2, etc. refer to fields. In external-table CSV expressions, VALUE:c1, VALUE:c2, etc. identify source positions.
- For this Parquet lab, staged $1 represents a structured row; the external table exposes it through VALUE.
- VALUE:countryOrRegion selects a named field. Field names are case-sensitive. ::VARCHAR converts to text.
- METADATA$FILENAME is a special column name; the dollar sign is part of its name.
- INT is an alias for NUMBER(38,0). Plain NUMBER also defaults to NUMBER(38,0). NUMBER(10,2) supports ten total digits, two after the decimal.
- Parquet stores data by column and supports efficient compression. It is often useful for large analytical datasets; there is no rule that large data requires CSV.
- Changing the order of column definitions does not change their field mappings. For Keyboard|1|1500, order_id must use VALUE:c2::INT.

## Refresh and independent copies
ALTER EXTERNAL TABLE ... REFRESH synchronizes external-table file metadata with storage. It does not refresh the stage, rewrite files, or load rows into a standard table.

Automatic refresh requires cloud event configuration; AUTO_REFRESH = TRUE alone is not the complete setup. Refresh is asynchronous. With AUTO_REFRESH = FALSE, run manual refresh or arrange scheduling.

An upstream process can change files, so results can change even though the external table itself is read-only. A standard table created using CREATE TABLE ... AS SELECT is a separate point-in-time copy, with no ongoing synchronization. Changes on either side do not automatically propagate to the other.

## Partitioning
Standard tables use Snowflake-managed micro-partitions. External tables instead track logical groups of external files in metadata.

Example Azure paths:
```text
orders/order_date=2026-10-01/orders_a.csv
orders/order_date=2026-10-02/orders_b.csv
```
The pipeline organizes files; the external-table definition extracts partition values from paths and declares PARTITION BY. Refresh registers partition membership. A matching partition filter can skip irrelevant files. This does not reorganize files or split a mixed-date CSV. The date in a path must accurately describe its contents. Our holidays lab does not configure partitions.

## Working Azure public-data lab
Use a role with the necessary object privileges and select an existing warehouse. Queries consume Snowflake compute. Run in order; do not rerun plain CREATE statements if those objects already exist.

```sql
CREATE DATABASE IF NOT EXISTS LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS LEARNING_DB.EXTERNAL_TABLE_LAB;
USE DATABASE LEARNING_DB;
USE SCHEMA EXTERNAL_TABLE_LAB;

CREATE FILE FORMAT IF NOT EXISTS holidays_parquet_format
  TYPE = PARQUET;

CREATE STAGE IF NOT EXISTS azure_holidays_stage
  URL = 'azure://azureopendatastorage.blob.core.windows.net/holidaydatacontainer/Processed/'
  FILE_FORMAT = (FORMAT_NAME = 'holidays_parquet_format');

LIST @azure_holidays_stage;

SELECT METADATA$FILENAME AS source_file, $1 AS file_row
FROM @azure_holidays_stage (PATTERN => '.*[.]parquet')
LIMIT 5;

CREATE EXTERNAL TABLE IF NOT EXISTS holidays_external
  LOCATION = @azure_holidays_stage
  REFRESH_ON_CREATE = TRUE
  AUTO_REFRESH = FALSE
  PATTERN = '.*[.]parquet'
  FILE_FORMAT = (FORMAT_NAME = 'holidays_parquet_format');

SELECT COUNT(*) AS total_rows FROM holidays_external;
ALTER EXTERNAL TABLE holidays_external REFRESH;

CREATE EXTERNAL TABLE holidays_external_typed (
  country VARCHAR AS (VALUE:countryOrRegion::VARCHAR),
  country_code VARCHAR AS (VALUE:countryRegionCode::VARCHAR),
  holiday_date DATE AS (VALUE:date::TIMESTAMP_NTZ::DATE),
  holiday_name VARCHAR AS (VALUE:holidayName::VARCHAR)
)
  LOCATION = @azure_holidays_stage
  REFRESH_ON_CREATE = TRUE
  AUTO_REFRESH = FALSE
  PATTERN = '.*[.]parquet'
  FILE_FORMAT = (FORMAT_NAME = 'holidays_parquet_format');

SELECT country, holiday_date, holiday_name
FROM holidays_external_typed
WHERE country_code = 'AR'
  AND holiday_date >= '2025-01-01'
  AND holiday_date < '2026-01-01'
ORDER BY holiday_date;

CREATE TABLE holidays_copy_demo AS
SELECT country, country_code, holiday_date, holiday_name
FROM holidays_external_typed;

DELETE FROM holidays_copy_demo WHERE country_code = 'AR';
SELECT COUNT(*) FROM holidays_copy_demo WHERE country_code = 'AR';
SELECT COUNT(*) FROM holidays_external_typed WHERE country_code = 'AR';
```

Confirmed by the learner: the filtered staged query worked; the external-table count was 6,955. The provided row contained countryOrRegion, countryRegionCode, date, holidayName, and normalizeHolidayName. The typed-table and copy-deletion exercise results have not been separately confirmed.

## Troubleshooting learned in this lab
An unfiltered staged query failed with “Parquet file size is 0 bytes.” An empty file still exists and has a filename; its contents cannot form a valid Parquet file. Filtering names ending in .parquet excluded the offending nonmatching file and resolved this lab's error. PATTERN filters names, not sizes or validity: an empty file named broken.parquet would still match.

| Symptom | Check |
|---|---|
| Access denied | Stage URL, Azure permissions, and active Snowflake role. |
| Missing new rows | LIST stage; verify location and PATTERN; refresh external-table metadata. |
| Misplaced values | CSV delimiter and field-position expressions. |
| Conversion errors | Source values versus declared types. |
| Different session fails | Exact error, CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA(). Normal external tables persist across sessions. |
| Slow scans | File sizes, layout, partitions, and query predicates. |

Do not assume a successful query proves every source record was read: Snowflake documents that file-scan errors can result in skipped files or partial results. Production pipelines need completeness checks, such as expected file and record counts.

## Review answers
1. New file missing: verify that it is visible and matches the table location/pattern, then refresh external-table metadata.
2. Why the pattern helped: it excluded non-Parquet-named files; it did not filter based on size or repair content.
3. Do new source files update the standard-table copy? No. A separate load or synchronization process is required.

## Remaining practice
- Private Azure storage integration and cloud permissions.
- Azure event notifications for automatic refresh.
- Partitioned Azure file layout and partition pruning verification.
- Confirm the typed-table and independent-copy exercise results.

## Sources
- [Snowflake external tables](https://docs.snowflake.com/en/user-guide/tables-external-intro)
- [CREATE EXTERNAL TABLE](https://docs.snowflake.com/en/sql-reference/sql/create-external-table)
- [Microsoft public holidays dataset](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-public-holidays)
- [Azure Data Lake Storage hierarchy](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-namespace)
