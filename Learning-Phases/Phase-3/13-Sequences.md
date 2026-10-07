# Snowflake: Sequences

## 1. What and why

A **sequence** is a schema-level object that generates numbers. I can use it to supply IDs instead of entering them manually. The same sequence can serve multiple tables.

```sql
-- Keep sequence examples in one schema.
USE DATABASE LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS SEQUENCE_LAB;
USE SCHEMA SEQUENCE_LAB;

-- Start at 1 and use a positive step of 1.
-- ORDER makes successive completed requests increase, but does not prevent gaps.
CREATE SEQUENCE order_id_seq
  START = 1
  INCREMENT = 1
  ORDER;

-- Generate a number. This consumes it; it is not a preview.
SELECT order_id_seq.NEXTVAL;
```

`START` and `INCREMENT` default to 1. The increment must be a non-zero integer; a negative increment generates values in the decreasing direction when ordered. Sequence values use the signed 64-bit integer range.

Run object creation and example inserts once. If objects already exist, inspect or reuse them rather than resetting a sequence that existing data depends on.

## 2. Automatically generate a table ID

```sql
-- DEFAULT requests a sequence value when the ID is omitted during insertion.
CREATE TABLE orders (
  order_id INT DEFAULT order_id_seq.NEXTVAL,
  product VARCHAR
);

-- Omit order_id so Snowflake evaluates its default expression.
INSERT INTO orders (product) VALUES ('Keyboard');

-- Inspect the saved ID. The earlier NEXTVAL value is not reused.
SELECT * FROM orders;
```

The default is not a rule that prevents manual IDs. An explicit value overrides it. Explicit NULL also does not mean “generate a number”; omit the column or use DEFAULT when generation is intended.

## 3. Why gaps happen

Sequences do not promise consecutive numbers. Gaps can occur when:

- I call NEXTVAL just to inspect a value.
- A transaction generates an ID and is later rolled back.
- A row using an ID is deleted.
- Snowflake allocates numbers that do not end up in stored rows.

An ordinary UPDATE of another column does not consume a sequence value. An operation explicitly requesting NEXTVAL, including a sequence-backed default, can do so.

**Use a sequence when gaps are acceptable.** It is not a row counter or a guaranteed gap-free invoice-numbering mechanism. Gaps are expected behavior, not a defect.

## 4. ORDER versus NOORDER

| Setting | Meaning | Illustrative output |
|---|---|---|
| ORDER | Successive completed requests increase with a positive increment. | 1, 2, 3, 4 |
| NOORDER | Requests may receive values out of numerical order. | 1, 101, 2, 102 |

These are examples, not promised outputs. Neither option prevents gaps or guarantees IDs follow business-event time. ORDER also does not assign IDs according to a desired row order in a bulk query.

NOORDER can improve performance when many requests generate IDs at once. I specify the setting explicitly instead of relying on account defaults. I use a timestamp column when I need to record when an event happened.

## 5. NEXTVAL generates; it does not look up

```sql
-- Each reference asks for another number, even within one SELECT.
-- Expected: first_id and second_id are different.
SELECT order_id_seq.NEXTVAL AS first_id,
       order_id_seq.NEXTVAL AS second_id;
```

Snowflake does not support sequence CURRVAL. If related rows need the same generated number, I generate it once and reuse the saved value.

## 6. One order with multiple products

The order row and all its item rows need the same order_id. Calling NEXTVAL separately for each row would generate different IDs.

```sql
-- Separate order details from the products belonging to each order.
CREATE TABLE customer_orders (order_id INT, customer VARCHAR);
CREATE TABLE order_items (order_id INT, product VARCHAR);

-- Generate one number and save it for this order.
-- Continue in the same session because this is a session variable.
SET saved_order_id = order_id_seq.NEXTVAL;

-- Keep the related table changes together: either commit all or roll them back.
BEGIN TRANSACTION;

-- Reading $saved_order_id reuses the value; it does not call NEXTVAL again.
INSERT INTO customer_orders VALUES ($saved_order_id, 'Anita');
INSERT INTO order_items VALUES
  ($saved_order_id, 'Keyboard'),
  ($saved_order_id, 'Mouse');

-- Commit only after both inserts succeed. If either fails, run ROLLBACK instead.
COMMIT;

-- Verify that both products have the same order ID as the customer order.
SELECT * FROM customer_orders;
SELECT * FROM order_items;
```

This is a small teaching example. For bulk pipelines, generate and carry IDs through the dataset rather than issuing one transaction per source row.

## 7. Generated IDs versus duplicate business data

A sequence supplies distinct generated values while its increment direction remains unchanged. It does not inspect existing table rows.

If I manually insert ID 500, the sequence does not automatically skip 500 later. Recreating a sequence from an old starting point or reversing its increment direction can also produce collisions with existing IDs. Standard Snowflake tables do not enforce primary-key uniqueness.

Two rows can have different generated IDs and still describe the same order. I need the source application's order ID to recognize that duplication.

## 8. Azure Parquet files → external table → generated IDs

```text
Application writes Parquet files to Azure Blob Storage
                         ↓
External stage points to the location
                         ↓
External table exposes the file rows
                         ↓
Load rows into a standard Snowflake table
                         ↓
Sequence supplies IDs for new destination rows
```

An external table is read-only. A sequence does not write permanent IDs into its Azure files. If I only need to query files, I can use IDs already supplied by the source instead of making a copy.

Parquet is a file format; the Azure files themselves are not Snowflake tables.

### Example input

| source_order_id | product | amount |
|---|---|---|
| WEB-501 | Keyboard | 1500 |
| WEB-502 | Mouse | 500 |

The following assumes an existing external table named `orders_external` with those three columns, in the current schema. The stage and Azure access are configured separately; this example starts at the loading step.

```sql
-- Create a separate generator for IDs stored in the destination.
CREATE SEQUENCE warehouse_order_seq START = 1 INCREMENT = 1 ORDER;

-- Preserve the application's ID as well as our generated numeric ID.
CREATE TABLE orders_loaded (
  warehouse_order_id INT DEFAULT warehouse_order_seq.NEXTVAL,
  source_order_id VARCHAR,
  product VARCHAR,
  amount NUMBER(10,2)
);

-- Register newly arrived files before reading the external table.
ALTER EXTERNAL TABLE orders_external REFRESH;

-- Initial-load example: omit the generated ID to invoke the default.
-- Repeating this INSERT would copy the same orders again with new IDs.
INSERT INTO orders_loaded (source_order_id, product, amount)
SELECT source_order_id, product, amount FROM orders_external;

-- Inspect the persistent IDs and copied values.
SELECT * FROM orders_loaded;
```

The Azure files are unchanged. The generated IDs exist in `orders_loaded`.

### Repeat loads without inserting existing orders again

```sql
-- Match using the application's stable ID, not a newly generated number.
-- Insert only source IDs missing from the destination.
MERGE INTO orders_loaded AS target
USING orders_external AS source
  ON target.source_order_id = source.source_order_id
WHEN NOT MATCHED THEN
  INSERT (source_order_id, product, amount)
  VALUES (source.source_order_id, source.product, source.amount);
```

This example assumes one source row per source_order_id and no overlapping loads. Duplicate source keys must be handled first. The example does not update existing orders; that requires an additional matched-update rule. It is not a universal duplicate-prevention guarantee.

**Do not generate a fresh ID in every SELECT and expect a permanent identity.** Save the generated value in a writable destination and preserve its relationship to the source key.

Writable Snowflake-managed Iceberg tables can also receive generated sequence values during inserts. An external table cannot store those new values back into its files. More Iceberg-specific setup is outside this topic.

## 9. Sequence versus IDENTITY / AUTOINCREMENT

A named sequence is a reusable object that multiple tables can reference. IDENTITY/AUTOINCREMENT provides automatic number generation attached to a table column. Neither is a promise of gap-free numbering, and manually entered values require care.

## 10. What I need to remember

- NEXTVAL consumes a new number.
- DEFAULT lets an INSERT omit the generated ID column.
- ORDER is not a promise of no gaps; NOORDER permits out-of-order values.
- Related rows reuse a generated value rather than calling NEXTVAL again.
- Generated-number uniqueness does not remove duplicate business records.
- Changing other columns does not automatically generate a new ID.
- External-file IDs become persistent only when written into a writable destination or supplied by the upstream file writer.

## Optional follow-up

- Bulk parent/child loading with GETNEXTVAL or multi-table INSERT.
- Role grants for sequence usage in production pipelines.
- Source deduplication and coordinated repeat loads using MERGE.

## References

- [Using sequences](https://docs.snowflake.com/en/user-guide/querying-sequences)
- [CREATE SEQUENCE](https://docs.snowflake.com/en/sql-reference/sql/create-sequence)
- [CREATE TABLE: defaults and identity columns](https://docs.snowflake.com/en/sql-reference/sql/create-table)
- [External-table column definitions](https://docs.snowflake.com/en/sql-reference/sql/create-external-table)

Reviewed: 7 October 2026.
