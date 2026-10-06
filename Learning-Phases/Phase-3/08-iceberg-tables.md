# Apache Iceberg tables: concepts and hands-on labs

Prepared: 6 October 2026. Azure is the cloud provider used throughout this guide.

## Learning and documentation requirements

- Explain new terminology before using it; keep explanations concrete.
- SQL examples must include comments explaining what each step does and why, with expected results where useful.
- Keep completed practice separate from exercises awaiting Azure access.
- Save this Markdown file in your GitHub learning repository. It contains placeholders, not credentials.

## 1. Picture Parquet first

Parquet is a file format: it defines how data is represented inside a file. One file can contain all the columns of a table for a subset of its rows. It does not require a separate file per column.

```text
orders.parquet — one file
  Row group 1: rows 1–3
    order_id: [1, 2, 3]
    product:  [Keyboard, Mouse, Monitor]
    amount:   [1500, 500, 9000]
  Row group 2: rows 4–6
    order_id: [4, 5, 6]
    product:  [Keyboard, Mouse, Monitor]
    amount:   [1500, 500, 9000]
  File metadata: describes the structure and locations of stored pieces
```

This is a conceptual picture, not the actual binary file representation. A **row group** is a batch of rows. A **column chunk** holds one column's values for that batch. A compatible engine querying only amount can avoid reading unrelated column chunks. Writers choose file and row-group sizes; three rows above is only an illustration.

For 1,000 columns and 1,000 rows, one possible layout is one file with ten row groups of 100 rows each, each group containing chunks for all 1,000 columns. Parquet also supports compression, which reduces stored size. It is useful for large analytical datasets, not just small datasets.

Reference: [Apache Parquet concepts](https://parquet.apache.org/docs/concepts/).

## 2. What Iceberg adds

Iceberg is an **open table format**: shared rules and metadata that let compatible engines treat files as a table. Parquet organizes values inside files; Iceberg tracks which files form a table version.

| Term | Meaning |
|---|---|
| Data files | Files holding actual rows; Snowflake Iceberg uses Parquet. |
| Metadata | Information describing schema, versions, and file membership. |
| Snapshot | A recorded table state; not necessarily a complete copied dataset. |
| Catalog | Maps a table name to current metadata and coordinates updates to that reference. |
| Commit | Makes a completed change visible as a consistent table state. |

Simplified update example:

```text
Old snapshot → files A + B
New snapshot → files A + C
```

A is reused. B can remain for retained history without being part of the current state. Readers follow metadata, rather than treating every file in a directory as current. Actual update strategies can differ; this is an explanatory example.

You issue SQL; Snowflake performs the underlying data and metadata operations for a Snowflake-managed table. Do not manually overwrite its files.

Reference: [Snowflake Iceberg concepts](https://docs.snowflake.com/en/user-guide/tables-iceberg).

## 3. Which table should you choose?

| Choice | Storage | Main distinction |
|---|---|---|
| Standard Snowflake table | Snowflake-managed | Native Snowflake format; straightforward for Snowflake-focused work. |
| Iceberg with Snowflake storage | Snowflake-managed | Open table format without setting up your own cloud storage. |
| Iceberg with Azure external storage | Your Azure account | Open format with files in your own lake. |
| External table over Azure files | Your Azure account | Read-only SQL access to referenced files. |

“Permanent” describes persistence/protection, while “Iceberg” describes table format; they are not opposites. Compatible external engines need configured catalog access and permissions—knowing a file URL is not sufficient interoperability setup.

**External stages** connect external tables to files. **External volumes** connect this Iceberg setup to its Azure storage. An arbitrary CSV or Parquet folder does not become a writable Iceberg table merely by referencing it.

Reference: [Iceberg storage choices](https://docs.snowflake.com/en/user-guide/tables-iceberg-storage).

## 4. Completed lab: Snowflake-provided storage

Status: learner confirmed DML and historical-query results. This is a replay script; do not rerun object creation or inserts against an existing completed lab without considering existing objects and duplicate rows. Select your existing warehouse in the worksheet first.

```sql
-- Choose an isolated schema so learning objects are easy to identify.
USE DATABASE LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS ICEBERG_LAB;
USE SCHEMA ICEBERG_LAB;

-- Use Iceberg while letting Snowflake provide both catalog and storage.
-- No Azure subscription, external stage, or custom external volume is needed.
CREATE ICEBERG TABLE orders_iceberg (
  order_id INT,
  product VARCHAR,
  amount NUMBER(10,2)
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'SNOWFLAKE_MANAGED';

-- Add three records to test writes to the Iceberg table itself.
INSERT INTO orders_iceberg VALUES
  (1, 'Keyboard', 1500.00),
  (2, 'Mouse', 500.00),
  (3, 'Monitor', 9000.00);

-- Modify one record, then remove another from the current table state.
UPDATE orders_iceberg SET amount = 700.00 WHERE order_id = 2;
DELETE FROM orders_iceberg WHERE order_id = 3;

-- Expected: order 1 = 1500; order 2 = 700; order 3 is absent.
SELECT * FROM orders_iceberg ORDER BY order_id;

-- Evolve the schema (table structure) by adding a business attribute.
-- Existing rows initially have NULL in the new column.
ALTER ICEBERG TABLE orders_iceberg ADD COLUMN order_status VARCHAR;
SELECT * FROM orders_iceberg ORDER BY order_id;

-- Supply values for the newly added attribute.
UPDATE orders_iceberg SET order_status = 'PLACED';

-- Retain one day of history for this Time Travel exercise.
ALTER ICEBERG TABLE orders_iceberg SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Run UPDATE and SET consecutively in the same session.
-- Save the UPDATE query ID so we can reference the state before it.
UPDATE orders_iceberg SET amount = 800.00 WHERE order_id = 2;
SET price_update_id = LAST_QUERY_ID();

-- Expected current amount: 800.
SELECT order_id, amount FROM orders_iceberg WHERE order_id = 2;

-- Expected historical amount: 700. This reads history; it does not undo anything.
-- $price_update_id references the session variable, unlike staged-file $1.
SELECT order_id, amount
FROM orders_iceberg BEFORE (STATEMENT => $price_update_id)
WHERE order_id = 2;
```

History is available only within retained history. Enabling retention does not recover already purged history. Schema evolution includes more than adding a column; type changes have compatibility restrictions and should be checked before production changes.

References: [Snowflake-storage lab](https://www.snowflake.com/en/developers/guides/get-started-snowflake-managed-iceberg-tables/), [Time Travel and retention](https://docs.snowflake.com/en/user-guide/tables-iceberg-metadata).

## 5. Future hands-on: Iceberg on external Azure storage

Status: documented for later practice; not executed against your Azure account.

### Goal and architecture

Create a new Iceberg table whose data and metadata files live in your Azure Blob Storage, while Snowflake manages the catalog and SQL writes. This does not move the table from section 4 automatically.

```text
Your SQL → Snowflake-managed Iceberg table/catalog
                         ↓
                  External volume
                         ↓
            Your private Azure container
               data + metadata files
```

### Prerequisites and values to collect

- Azure subscription and permission to create a storage account/container.
- An Azure administrator who can grant application consent and assign storage roles.
- A Snowflake administrator for external-volume setup; a working warehouse for queries.
- Use an empty, dedicated learning storage account to keep permissions and cleanup isolated.
- Azure storage and Snowflake compute incur usage costs; this is not the public read-only dataset lab.

| Placeholder | Replace with |
|---|---|
| `<storage_account>` | Globally unique Azure storage account name |
| `<tenant_id>` | Microsoft Entra tenant ID, not subscription ID |
| `<warehouse_name>` | Your existing Snowflake warehouse |

### Step A — Prepare Azure

In Azure Portal, create a dedicated general-purpose v2 storage account, preferably in the Snowflake account's Azure region. For this Blob lab, leave hierarchical namespace disabled. Create a private container named `iceberg-lab`; do not upload sample files. Find your tenant ID under Microsoft Entra ID → Overview.

Allow the required network path from Snowflake. A private container means authenticated data access; it does not by itself require a private network endpoint. If storage firewall restrictions apply, follow Snowflake's Azure networking instructions rather than assuming IAM alone is sufficient.

### Step B — Define the external volume

Replace placeholders before running. The volume is account-level, not a table or schema object.

```sql
-- Use administrative access for one-time integration setup.
USE ROLE ACCOUNTADMIN;

-- Describe the Azure location Snowflake will use for Iceberg files.
-- ALLOW_WRITES permits writes in Snowflake but does not grant Azure permissions.
CREATE EXTERNAL VOLUME AZURE_ICEBERG_VOL
  STORAGE_LOCATIONS = (
    (
      NAME = 'azure_iceberg_lab_location'
      STORAGE_PROVIDER = 'AZURE'
      STORAGE_BASE_URL = 'azure://<storage_account>.blob.core.windows.net/iceberg-lab/'
      AZURE_TENANT_ID = '<tenant_id>'
    )
  )
  ALLOW_WRITES = TRUE;

-- Retrieve the Azure consent link and application identity for the next step.
DESC EXTERNAL VOLUME AZURE_ICEBERG_VOL;
```

Reference: [CREATE EXTERNAL VOLUME](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume).

### Step C — Authorize the identity in Azure

1. From the description output, copy `AZURE_CONSENT_URL` and `AZURE_MULTI_TENANT_APP_NAME`.
2. Open the consent URL with an appropriately privileged Azure account and accept.
3. In the storage account's **Access control (IAM)**, add **Storage Blob Data Contributor** for the Snowflake application identity. Search using the app-name prefix before the underscore if necessary.
4. Assign at **storage-account scope** for this documented setup; container-only assignment can miss the account-level delegation-key permission.
5. Allow role propagation. The application identity may take an hour or longer to appear.

A service principal is an application's identity in your Azure tenant. Consent establishes the application access relationship; the role grants storage operations. Both are needed. This guide follows [Snowflake's Azure external-volume procedure](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-external-volume-azure).

### Step D — Verify before creating a table

```sql
-- Test authentication and storage operations before troubleshooting table SQL.
-- With writes enabled, verification includes writing, reading, listing,
-- and deleting a test file. Inspect the returned result for failures.
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('AZURE_ICEBERG_VOL');
```

Proceed only when verification succeeds. Reference: [Verification function](https://docs.snowflake.com/en/sql-reference/functions/system_verify_external_volume).

### Step E — Create the Azure-backed table

For simplicity, this personal lab continues under the setup role. Production should use a separate role with only required database/schema, warehouse, table, and external-volume privileges.

```sql
-- Select compute and keep this exercise separate from the earlier lab.
USE WAREHOUSE <warehouse_name>;
USE DATABASE LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS ICEBERG_AZURE_LAB;
USE SCHEMA ICEBERG_AZURE_LAB;

-- Snowflake manages the catalog; your external volume supplies Azure storage.
-- BASE_LOCATION is a relative table path, not a full Azure URL.
CREATE ICEBERG TABLE orders_azure (
  order_id INT,
  product VARCHAR,
  amount NUMBER(10,2)
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'AZURE_ICEBERG_VOL'
  BASE_LOCATION = 'orders_azure';

-- Inspect the registered Iceberg table to confirm catalog and volume settings.
SHOW ICEBERG TABLES LIKE 'ORDERS_AZURE';

-- Populate the new table: Snowflake writes its files to Azure.
INSERT INTO orders_azure VALUES
  (1, 'Keyboard', 1500.00),
  (2, 'Mouse', 500.00),
  (3, 'Monitor', 9000.00);

-- Read the initial values before testing changes.
SELECT * FROM orders_azure ORDER BY order_id;

-- Update and delete through Iceberg, keeping data and metadata consistent.
UPDATE orders_azure SET amount = 700.00 WHERE order_id = 2;
DELETE FROM orders_azure WHERE order_id = 3;

-- Expected: two rows, with amounts 1500 and 700.
SELECT * FROM orders_azure ORDER BY order_id;
```

Reference: [CREATE ICEBERG TABLE with Snowflake catalog](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-snowflake).

### Step F — Inspect Azure and understand what changed

Open the container in Azure Portal and refresh its listing. Inspect the generated paths for data and metadata. Snowflake can append generated identifiers to table directories; do not require an exact literal `orders_azure/data/` path. File counts and visibility timing can vary.

Parquet is binary: Azure's text preview is not a table viewer. Use SQL to validate rows. A removed order may still exist in older retained files; current queries follow the current committed state. Never manually remove older-looking files while the table is active.

Storage layout reference: [External-volume file organization](https://docs.snowflake.com/en/user-guide/tables-iceberg-managing-external-volumes).

### Validation checklist

- [ ] External-volume verification succeeds.
- [ ] SHOW ICEBERG TABLES identifies the intended volume and Snowflake catalog.
- [ ] Azure contains generated data/metadata files.
- [ ] Initial query returns three rows.
- [ ] Final query returns two rows; order 2 has amount 700.
- [ ] No external stage or CSV file format was needed to manage this table.

### Troubleshooting

| Symptom | Investigate |
|---|---|
| App identity not found | Consent tenant, returned app name, propagation delay. |
| Authorization/delegation-key failure | Correct principal, storage-account role scope, role propagation. |
| Cannot write or delete | ALLOW_WRITES and Azure data permissions. |
| Network failure | Storage firewall and Snowflake connectivity path. |
| Wrong location or no expected files | Account/container spelling, volume selection, generated subpaths. |
| CREATE succeeds but SELECT fails | Active warehouse and Snowflake role privileges. |
| Object already exists | Reuse it or choose a new lab name; do not blindly replace it. |

### Operational limits and cleanup

This lab demonstrates Snowflake as the sole writer/catalog manager. Other engines need supported catalog integration and permissions; do not attach an unrelated writer directly to these paths.

External-volume storage does not receive Snowflake Fail-safe protection. Plan cloud recovery/versioning and retention deliberately. Avoid Azure lifecycle rules that delete active or retained Iceberg files. Snowflake-provided storage has different protection behavior. See [storage choices](https://docs.snowflake.com/en/user-guide/tables-iceberg-storage).

After finishing, retain the lab if needed for future study. For disposable resources, drop only this lab table, then its external volume after all dependent tables are gone. Do not assume DROP immediately purges retained Azure data. Inspect remaining storage and retention before deleting the dedicated Azure resources.

```sql
-- OPTIONAL CLEANUP: run only when this lab is no longer needed.
-- Remove the table before the volume because the table depends on it.
DROP ICEBERG TABLE LEARNING_DB.ICEBERG_AZURE_LAB.ORDERS_AZURE;

-- Remove only the dedicated lab volume, once it has no dependent tables.
DROP EXTERNAL VOLUME AZURE_ICEBERG_VOL;
```

## 6. Review questions

1. Why does a Parquet folder alone not provide a complete Iceberg table?
2. What is the difference between the catalog and an external volume?
3. Does ALLOW_WRITES = TRUE grant Azure permissions?
4. Why might old files remain after a SQL DELETE?
5. Why do two tables using Snowflake storage still differ if one is standard and one is Iceberg?

<details>
<summary>Answers</summary>

1. Iceberg needs metadata describing schema, versions, and current file membership.
2. The catalog resolves table metadata; the volume configures access to storage.
3. No. Azure identity authorization is also required.
4. Retained history can reference them; deletion from current state is not immediate physical erasure.
5. Their underlying table/storage formats differ; Iceberg provides an open format with supported cross-engine access.

</details>

## 7. Learning status

- Completed: Parquet structure, snapshots/catalog fundamentals, storage choices, SQL writes, adding a column, and Time Travel lab.
- Prepared, not executed: external Azure storage lab above.
- Still to study: Iceberg partition transforms and evolution, detailed maintenance/compaction, and external-catalog interoperability. This guide is not a claim that all production Iceberg features have been covered.
