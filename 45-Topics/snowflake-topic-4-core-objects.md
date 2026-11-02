# Snowflake Notes: Topic 4, Core Objects

## 1. Object hierarchy

Organization > Account > **Database** > **Schema** > objects (tables, views, stages, file formats, sequences, and more).

- **Database:** top-level container for related data (for example `SALES_DB`).
- **Schema:** a folder inside a database that groups objects (for example `RAW`, `CLEAN`, `REPORTING`).
- Every database automatically gets two schemas: **PUBLIC** (the default) and **INFORMATION_SCHEMA** (read-only metadata about the database).

*Example:* `SALES_DB` has schemas `RAW` (data as loaded), `CLEAN` (cleaned data), and `REPORTING` (views for dashboards).

## 2. Tables and views

A **table** stores rows and columns (the actual data, in micro-partitions).

A **view** stores only a query. Nothing is copied. Each time you query it, Snowflake runs that query on the underlying tables, so you always see fresh data.
- Pros: always fresh, no extra storage, hides complexity, can limit which columns people see.
- Con: runs the full query every time, so a heavy view is slow.

## 3. Table types

| | Permanent | Transient | Temporary | External |
|---|---|---|---|---|
| Created with | `CREATE TABLE` | `CREATE TRANSIENT TABLE` | `CREATE TEMPORARY TABLE` | `CREATE EXTERNAL TABLE` |
| Lives until | You drop it | You drop it | The **session ends** | You drop it |
| Visible to | Anyone with access | Anyone with access | Only the creating session | Anyone with access |
| Time Travel | 1 day (Standard), up to 90 (Enterprise+) | 0 or 1 day only | 0 or 1 day only | None |
| Fail-safe | **7 days** | **None** | **None** | None |
| Where data lives | Snowflake storage | Snowflake storage | Snowflake storage | **Your external storage** |
| Typical use | Production data | Staging, scratch, easily rebuilt data | Temporary work inside one session or script | Querying files in a data lake without loading them |

**External tables:** the data stays outside Snowflake, and Snowflake stores only metadata about the files. They are read-only (no INSERT, UPDATE, or DELETE) and usually slower than normal tables.

*Scenarios*
- Customer and orders tables: **permanent**.
- A daily staging table reloaded every night from files: **transient** (no need to pay for Fail-safe storage).
- Intermediate results while writing one long script: **temporary**.
- Raw Parquet files sitting in Azure storage that you want to peek at: **external**.

## 4. Views: standard, materialized, secure

**Standard view:** stores only the query.

**Materialized view:** actually **stores the result** of the query and keeps it updated.
- Snowflake refreshes it automatically in the background, which costs credits.
- Much faster for expensive aggregations that are queried often.
- Needs Enterprise edition or higher. It can use only **one table** (no joins), with other limits.
- *Example:* a dashboard repeatedly asks for total sales per day over a billion-row table. A materialized view keeps the daily totals ready.

**Secure view:** hides the view's definition (the query text) from anyone except the owner, and prevents the optimizer from shortcuts that could accidentally leak data the viewer shouldn't see.
- Required when sharing data with other accounts through data sharing.
- Slightly slower, because some optimizations are turned off.
- *Example:* a partner should see only their own region's rows. You build a secure view filtered by region and share it, so they can't read the filter logic or reach other rows.

## 5. Object naming and fully qualified names

**Naming rules (unquoted identifiers)**
- Start with a letter or underscore. After that: letters, digits, underscores, `$`.
- No spaces or special characters.
- **Not case-sensitive.** `orders`, `Orders`, and `ORDERS` are the same, and Snowflake stores it as `ORDERS`.
- Maximum 255 characters.

**Quoted identifiers:** `"My Orders"` allows spaces and special characters, but is **case-sensitive**, so you must type it exactly, with quotes, every time. Avoid them in practice.

**Fully qualified name:** `database.schema.object`

```sql
SELECT * FROM SALES_DB.RAW.ORDERS;
```

Or set your context and use the short name:
```sql
USE DATABASE SALES_DB;
USE SCHEMA RAW;
SELECT * FROM ORDERS;
```

*Why it matters:* the same table name can exist in many schemas. `RAW.ORDERS` and `CLEAN.ORDERS` are two different tables. Fully qualified names avoid querying the wrong one.

## 6. What "retention" means

**Retention** means: *how long does Snowflake keep old versions of your data after you change or delete it?* This lets you recover from mistakes.

Two kinds:
- **Time Travel:** you can query or restore the old data yourself (for example, "show me this table as it was 30 minutes ago", or `UNDROP TABLE`).
- **Fail-safe:** an extra 7 days after Time Travel ends, where **only Snowflake support** can recover data.

"Compare retention behavior" means: check how long each table type keeps old data and whether it has Fail-safe.

---

## Quick recap
1. Hierarchy: database > schema > objects. Every database has PUBLIC and INFORMATION_SCHEMA.
2. Table = stores data. View = stores a query. Materialized view = stores the result and refreshes it. Secure view = hides the definition and protects data when sharing.
3. Permanent: Fail-safe 7 days, longest Time Travel. Transient: no Fail-safe, 1 day max. Temporary: session only. External: data stays outside Snowflake, read-only.
4. Unquoted names are case-insensitive and stored uppercase. Fully qualified: `database.schema.object`.
5. Retention = how long old data is recoverable (Time Travel by you, Fail-safe by Snowflake only).

---

## Hands-on: create a database with 3 schemas, one table of each type, and compare retention (do later on a laptop)

### Step 1: create the objects
```sql
CREATE OR REPLACE DATABASE retention_lab;

CREATE SCHEMA retention_lab.perm_sch;
CREATE TRANSIENT SCHEMA retention_lab.trans_sch;
CREATE SCHEMA retention_lab.temp_sch;

CREATE TABLE retention_lab.perm_sch.emp_perm (id INT, name STRING);
CREATE TRANSIENT TABLE retention_lab.trans_sch.emp_trans (id INT, name STRING);
CREATE TEMPORARY TABLE retention_lab.temp_sch.emp_temp (id INT, name STRING);

INSERT INTO retention_lab.perm_sch.emp_perm VALUES (1,'Asha'),(2,'Ravi');
INSERT INTO retention_lab.trans_sch.emp_trans VALUES (1,'Asha'),(2,'Ravi');
INSERT INTO retention_lab.temp_sch.emp_temp VALUES (1,'Asha'),(2,'Ravi');
```
(`CREATE TRANSIENT SCHEMA` makes every table created inside it transient by default.)

### Step 2: compare retention days
```sql
SHOW TABLES IN DATABASE retention_lab;
```
Look at the `kind` and `retention_time` columns (names may differ slightly). Write down what each table shows.

### Step 3: try to push the limits
```sql
ALTER TABLE retention_lab.trans_sch.emp_trans SET DATA_RETENTION_TIME_IN_DAYS = 2;
```
This should **fail**, because transient tables allow at most 1 day. Now the permanent table:
```sql
ALTER TABLE retention_lab.perm_sch.emp_perm SET DATA_RETENTION_TIME_IN_DAYS = 5;
```
This works only on Enterprise edition or higher (your trial account may be Enterprise). On Standard it fails, since the limit is 1 day.

### Step 4: test recovery
```sql
DROP TABLE retention_lab.perm_sch.emp_perm;
UNDROP TABLE retention_lab.perm_sch.emp_perm;
```
Repeat with `emp_trans`. Both should come back, because both have Time Travel (1 day).

### Step 5: test temporary behavior
Close the worksheet (or sign out and back in) and open a **new** one. Then run:
```sql
SELECT * FROM retention_lab.temp_sch.emp_temp;
```
It should fail with "does not exist", because a temporary table disappears when its session ends.

### Step 6: see Fail-safe (the next day)
```sql
SELECT table_name, active_bytes, time_travel_bytes, failsafe_bytes
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE table_catalog = 'RETENTION_LAB';
```
This view updates with a delay of a few hours. You should see `failsafe_bytes` for the permanent table, and none for transient and temporary.

### Write your comparison table
| | Permanent | Transient | Temporary |
|---|---|---|---|
| Max Time Travel | ? | ? | ? |
| Fail-safe | ? | ? | ? |
| Survives new session? | ? | ? | ? |
| Can UNDROP? | ? | ? | ? |

### Clean up
```sql
DROP DATABASE retention_lab;
```
