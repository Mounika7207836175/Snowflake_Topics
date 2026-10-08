# Snowflake: File Formats

## 1. What is a file format?

A **file format** is a reusable set of instructions that tells Snowflake how to read input files or write output files.

It does **not** store data. It does **not** define a table's schema. A table defines its columns and data types; a file format explains file details such as delimiters, headers, quotes, null text, compression, and date/timestamp representation.

```text
CSV / JSON / Parquet file
            ↓
       File format
            ↓
Snowflake can interpret the file correctly
            ↓
Stage query, COPY INTO, or external table
```

File formats are schema-level objects. A named file format can be reused by stages, COPY INTO commands, and external-table definitions.

## 2. CSV example

Source file:

```csv
product_id,product_name,price
1,"Keyboard, wired",1500
2,Mouse,500
```

```sql
-- Work in the schema used for stage examples.
USE DATABASE LEARNING_DB;
USE SCHEMA STAGE_LAB;

-- Create reusable rules for reading CSV files.
-- The quote setting preserves "Keyboard, wired" as one field.
CREATE FILE FORMAT products_csv_format_v2
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', '');

-- Inspect the stored configuration.
DESC FILE FORMAT products_csv_format_v2;

-- Apply the rules while querying files directly from a stage.
-- $1, $2, and $3 refer to fields 1, 2, and 3 in each CSV record.
SELECT
  $1::INT AS product_id,
  $2::VARCHAR AS product_name,
  $3::NUMBER(10,2) AS price
FROM @products_stage
  (FILE_FORMAT => 'products_csv_format_v2');
```

Result verified:

| PRODUCT_ID | PRODUCT_NAME | PRICE |
|---|---|---:|
| 1 | Keyboard | 1500.00 |
| 2 | Mouse | 500.00 |

## 3. Important CSV settings

| Setting | Example | What it does |
|---|---|---|
| `TYPE` | `CSV` | Specifies a delimited text file. CSV can use separators other than commas. |
| `FIELD_DELIMITER` | `','` or `'|'` | Splits one record into fields. |
| `RECORD_DELIMITER` | Newline by default | Separates records/rows. |
| `SKIP_HEADER` | `1` | Skips header rows during a load or query. |
| `FIELD_OPTIONALLY_ENCLOSED_BY` | `'"'` | Supports fields enclosed in quotes. |
| `NULL_IF` | `('NULL', '')` | Converts listed file text values into SQL `NULL`. |
| `COMPRESSION` | `AUTO`, `GZIP`, `NONE` | Explains whether/how the file is compressed. |
| `DATE_FORMAT` | `'YYYY-MM-DD'` | Specifies how date text should be interpreted. |
| `TIMESTAMP_FORMAT` | `'YYYY-MM-DD HH24:MI:SS'` | Specifies how timestamp text should be interpreted. |
| `ENCODING` | `UTF8` | Specifies the file's character encoding. |

## 4. SKIP_HEADER versus PARSE_HEADER

These settings are different:

| Setting | What it does |
|---|---|
| `SKIP_HEADER = 1` | Ignores the first physical row. It does not create column names. This is common when loading into an existing table. |
| `PARSE_HEADER = TRUE` | Reads header values as column names. This is useful for schema detection and some schema-evolution workflows. |

For the `products` table, `SKIP_HEADER = 1` is appropriate because the destination columns already exist.

## 5. Direct-stage fields versus external-table fields

For a CSV record `1,Keyboard,1500`:

| Query source | Field syntax |
|---|---|
| Direct stage query: `FROM @products_stage` | `$1`, `$2`, `$3` |
| CSV external table query: `FROM products_external` | `VALUE:c1`, `VALUE:c2`, `VALUE:c3` |

This difference comes from querying a stage versus querying an external table. It is not about internal versus external storage.

```sql
-- Direct stage query: field 2 is Keyboard.
SELECT $2::VARCHAR AS product_name FROM @products_stage;

-- External-table query: c2 is the same source field.
-- products_external is an example object and must exist first.
SELECT VALUE:c2::VARCHAR AS product_name FROM products_external;
```

## 6. Other supported file types

Snowflake supports named file formats for `CSV`, `JSON`, `AVRO`, `ORC`, `PARQUET`, and `XML`.

### JSON

JSON is semi-structured text. It commonly loads into a `VARIANT` column first, then fields are extracted using paths.

```sql
-- JSON can contain one document, multiple documents, or an outer array.
CREATE FILE FORMAT events_json_format
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE;
```

`STRIP_OUTER_ARRAY = TRUE` treats each object inside a top-level JSON array as a separate row.

### Parquet

Parquet is a binary, column-oriented format that carries structural information. It usually needs fewer settings than CSV.

```sql
-- Parquet provides its own column structure and logical types.
CREATE FILE FORMAT orders_parquet_format
  TYPE = PARQUET;
```

The earlier public Azure holidays dataset used this type.

## 7. Named versus inline file formats

### Named format

Use a named format for repeatable pipelines because the rules have one reusable name.

```sql
-- Reuse the saved configuration by name during loading.
COPY INTO products
FROM @products_stage
FILE_FORMAT = (FORMAT_NAME = 'products_csv_format_v2');
```

### Inline format

Use an inline format for a small, one-time task.

```sql
-- Define the reading rules only for this COPY command.
COPY INTO products
FROM @products_stage
FILE_FORMAT = (
  TYPE = CSV
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
);
```

## 8. Validate before loading

Query staged files first when possible. This catches delimiter, header, quote, and type-conversion issues before rows are inserted into a table.

```sql
-- Ask Snowflake to validate this staged CSV without loading rows.
-- Return up to 10 detected errors.
COPY INTO products
FROM @products_stage
FILES = ('products.csv')
FILE_FORMAT = (FORMAT_NAME = 'products_csv_format_v2')
VALIDATION_MODE = 'RETURN_10_ROWS';
```

`VALIDATION_MODE` validates instead of performing the load. Remove it to execute the actual COPY command.

```sql
-- Load only when validation is clear.
-- Abort the entire load if Snowflake finds a row error.
COPY INTO products
FROM @products_stage
FILES = ('products.csv')
FILE_FORMAT = (FORMAT_NAME = 'products_csv_format_v2')
ON_ERROR = 'ABORT_STATEMENT';
```

Avoid using `ON_ERROR = 'CONTINUE'` casually in production. It can load partial data while skipping bad rows. If it is required, capture and monitor rejected records.

## 9. Common problems

| Symptom | Likely cause | First check |
|---|---|---|
| Everything appears in one column | Wrong `FIELD_DELIMITER`. | Inspect the source separator. |
| Header is loaded as a row | `SKIP_HEADER` is missing or incorrect. | Inspect the first line. |
| A quoted comma splits a field | Quote setting is missing. | Set `FIELD_OPTIONALLY_ENCLOSED_BY`. |
| Text such as `NULL` appears instead of a SQL null | `NULL_IF` is not configured. | Confirm the source's null representation. |
| Date/timestamp conversion fails | Source layout differs from the format setting. | Set an explicit date/timestamp format. |
| Garbled text | Incorrect encoding. | Confirm source encoding, often UTF-8. |
| Load errors or wrong values | File format and actual file content do not match. | Query/validate the stage before COPY. |

## 10. What I need to remember

- A file format is reading/writing instructions, not data or table schema.
- Stages hold files; file formats explain those files; COPY INTO loads rows.
- CSV needs attention to delimiters, headers, quotes, null text, and compression.
- `SKIP_HEADER` skips a row; `PARSE_HEADER` reads column names.
- Use named formats for reusable pipelines and inline formats for one-time work.
- Validate files before production loads.

## Optional follow-up

- JSON loading into VARIANT and field extraction.
- Schema detection and schema evolution using CSV/Parquet headers or structures..
- Error capture and load-history monitoring.

## References

- [CREATE FILE FORMAT](https://docs.snowflake.com/en/sql-reference/sql/create-file-format)
- [Preparing data files](https://docs.snowflake.com/en/user-guide/data-load-considerations-prepare)
- [COPY INTO table](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)

Reviewed: 7 October 2026.
