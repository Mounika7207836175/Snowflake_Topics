# Phase 3 | Topic 1: Databases and Schemas (Management Depth)

Core concepts (hierarchy, namespace boundaries, INFORMATION_SCHEMA) were covered in depth in Phase 1 Topics 3, 6, and 9. This topic adds the practical, day-to-day management operations.

## Creating With Useful Options
```sql
CREATE DATABASE IF NOT EXISTS SALES_DB
  COMMENT = 'Main sales data warehouse database';
```
- `IF NOT EXISTS` — prevents an error if the database already exists; command does nothing instead of failing. Useful in scripts that might run multiple times.
- `COMMENT` — human-readable description attached to the object, visible in `SHOW DATABASES` and `INFORMATION_SCHEMA` — helps teams understand an object's purpose without asking.

## Renaming and Dropping, With a Safety Net
```sql
ALTER DATABASE SALES_DB RENAME TO SALES_DATA_DB;
DROP DATABASE SALES_DATA_DB;
```
**Critical detail:** `DROP DATABASE` doesn't destroy data instantly. Because of **Time Travel**, a dropped database can be **undropped** within the retention window:
```sql
UNDROP DATABASE SALES_DATA_DB;
```
A genuinely important safety feature — an accidental `DROP` isn't necessarily catastrophic if caught in time.

**Limit on recovery:** bound by the Time Travel retention period (1 day on Standard, up to 90 days on Enterprise+). Once that window passes, the data moves into **Fail-safe** and is no longer self-service recoverable — only Snowflake support could potentially help, for a limited additional period.

**Interview angle:** *"If you accidentally drop a database, is the data gone forever?"*
Not necessarily — `UNDROP DATABASE` can recover it, provided it's within the Time Travel retention period.

## Cloning (Zero-Copy Cloning)
```sql
CREATE DATABASE SALES_DB_DEV CLONE SALES_DB;
```
Creates an instant, full copy of the entire database (all schemas, tables, data) **without duplicating the actual storage**. This is how real teams create a DEV copy of PROD data in seconds, at near-zero initial storage cost. (Deep mechanics of *how* this works — covered in a dedicated later topic.)

## Hands-On: Testing Undrop and Clone

**Undrop test:**
```sql
CREATE DATABASE IF NOT EXISTS TEST_UNDO_DB COMMENT = 'Testing undrop behavior';
DROP DATABASE TEST_UNDO_DB;
SHOW DATABASES LIKE 'TEST_UNDO_DB';     -- should return no rows
UNDROP DATABASE TEST_UNDO_DB;
SHOW DATABASES LIKE 'TEST_UNDO_DB';     -- should show it again
```
**Result:** confirmed working — database successfully recovered after drop.

**Clone test:**
```sql
CREATE DATABASE SALES_DB_CLONE CLONE SALES_DB;
SHOW DATABASES LIKE 'SALES_DB_CLONE';
SELECT * FROM SALES_DB_CLONE.RAW.ORDERS_RAW;
```
**Result:** confirmed — querying the cloned database's table returned the exact same data as the original `ORDERS_RAW`, proving the clone fully and instantly copied everything.

---
**Next Topic:** Standard tables
