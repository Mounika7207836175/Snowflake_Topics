# Parquet File Format in Snowflake

## 1. What is Parquet?

Apache Parquet is an open-source binary columnar file format designed for efficient storage and analytical processing. It is widely used in data lakes and distributed data platforms.

## 2. Parquet internal structure

A Parquet file is organized into row groups. Each row group contains column chunks, and each column chunk contains pages.

```text
Parquet file
  File header
  Row group 1
    Column chunk: employee_id
    Column chunk: name
    Column chunk: salary
  Row group 2
    Column chunk: employee_id
    Column chunk: name
    Column chunk: salary
  File metadata/footer
```

## 3. Why Parquet is useful

- Columnar layout can reduce reads when queries need only selected columns.
- Compression and encoding can reduce storage and I/O.
- Metadata describes the schema.
- It supports nested data structures.
- It is widely supported by data lake and analytics tools.

## 4. Parquet in Snowflake

Create a file format:

```sql
CREATE FILE FORMAT PARQUET_FORMAT
    TYPE = PARQUET;
```

Load staged data:

```sql
COPY INTO MY_PARQUET_TABLE
FROM @MY_STAGE
FILE_FORMAT = (FORMAT_NAME = 'PARQUET_FORMAT');
```

Parquet can contain nested structures. Depending on the schema and loading approach, data may be loaded into a `VARIANT` column and queried using path expressions.

Inspect staged data:

```sql
SELECT $1
FROM @MY_STAGE
(FILE_FORMAT => 'PARQUET_FORMAT');
```

Use the correct stage query syntax and file format reference for the selected Snowflake object.

## 5. Row groups and column chunks

A row group contains a subset of rows. Each column chunk stores the values for one column within that row group. Pages are smaller units within a column chunk.

This structure allows readers to locate and read relevant columns.

## 6. Practical workflow

1. Put the Parquet file in a stage.
2. Create a Parquet file format.
3. Confirm the file using `LIST`.
4. Inspect schema and nested structures.
5. Select a target table design.
6. Load with `COPY INTO`.
7. Validate row counts, types and values.

## 7. Key points

- Parquet is binary and columnar.
- A row group contains column chunks.
- Column chunks contain pages.
- Parquet supports compression, encoding and nested structures.
- Snowflake file format type is `PARQUET`.

## 8. Practice

1. Create a Parquet file format.
2. List a Parquet file in a stage.
3. Inspect the file's structure.
4. Load it into an appropriate table.
5. Explore nested values and validate the result.
