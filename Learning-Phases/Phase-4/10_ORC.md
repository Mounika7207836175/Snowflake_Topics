# ORC File Format in Snowflake

## 1. What is ORC?

ORC stands for Optimized Row Columnar. It is a binary columnar file format designed for efficient storage and analytical processing.

Instead of storing every field of one record together, ORC organizes values by columns within groups of rows.

## 2. ORC internal structure

An ORC file is organized into stripes. A stripe contains data for a group of rows and includes column streams, index information and stripe metadata. The file footer stores metadata that helps readers understand the file.

```text
ORC file
  Stripe 1
    Index data
    Column data streams
    Stripe footer
  Stripe 2
    Index data
    Column data streams
    Stripe footer
  File footer and metadata
```

## 3. Why columnar storage helps

Suppose a table has 30 columns but a query needs only `employee_id` and `salary`. A columnar reader can focus on the required column streams instead of reading every column's values.

This can reduce I/O for analytical queries.

## 4. Compression and encoding

ORC supports compression and column-aware encoding. Similar values in a column can often be compressed efficiently. This can reduce storage size and data read.

## 5. ORC in Snowflake

Create a file format:

```sql
CREATE FILE FORMAT ORC_FORMAT
    TYPE = ORC;
```

List staged files:

```sql
LIST @MY_STAGE;
```

Load staged ORC data:

```sql
COPY INTO MY_TABLE
FROM @MY_STAGE
FILE_FORMAT = (FORMAT_NAME = 'ORC_FORMAT');
```

Choose the target table design based on the source schema and how the data should be queried. Inspect semi-structured output where appropriate.

## 6. Practical workflow

1. Place an ORC file in an internal or external stage.
2. Create an ORC file format.
3. Use `LIST` to confirm the file exists.
4. Inspect the file schema and data.
5. Choose the target table structure.
6. Load with `COPY INTO`.
7. Validate row counts, types and values.

## 7. Key points

- ORC is binary, not directly human-readable text.
- ORC is columnar.
- Stripes group rows and contain column streams.
- Compression and encoding can improve storage efficiency.
- Snowflake file format type is `ORC`.

## 8. Practice

1. Create an ORC file format.
2. List an ORC file in a stage.
3. Load the file into a suitable target table.
4. Validate row counts and values.
5. Inspect nested or complex data if present.
