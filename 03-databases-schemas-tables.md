# Topic 3: Databases, Schemas, and Tables — Snowflake's Object Hierarchy

## Object Hierarchy — Complete Picture

```
Organization → Account → Database → Schema → Objects (Tables, Views, Stages, File Formats, Sequences, Streams, Tasks, Procedures, Functions)
```

**Important:** Warehouses, Users, and Roles are NOT inside databases — they live at the **account level**, independent of any database/schema. A warehouse is compute, not data, so it sits outside the data hierarchy entirely.

## Simple Mental Model
- **Account** = the whole house (your entire Snowflake setup)
- **Database** = a room in the house (e.g., "Kitchen," "Bedroom")
- **Schema** = a cupboard inside that room
- **Table** = a box inside the cupboard, where actual rows/columns of data live

Full address to query anything: `DATABASE.SCHEMA.TABLE`
```sql
SELECT * FROM SALES_DB.ANALYTICS.DAILY_SALES;
```

---

## Databases — Deeper Look
A database is a **logical namespace boundary**. Two different databases can have identically-named schemas/tables with zero conflict, because the full path disambiguates everything.

System database to know: `SNOWFLAKE` — Snowflake's own metadata/usage database (query history, credit usage, account usage views — critical for cost monitoring).

Every database auto-generates an `INFORMATION_SCHEMA` — built-in metadata views about every object in that database (tables, columns, privileges, etc.):
```sql
SELECT * FROM SALES_DB.INFORMATION_SCHEMA.TABLES;
```
Returns metadata about every table in `SALES_DB` — size, row count, creation time, etc. Used to programmatically audit/discover objects instead of manually browsing.

---

## Schemas — Deeper Look
A schema is the **security and organization boundary** below the database.

- Every schema also gets its own auto-created `INFORMATION_SCHEMA`.
- Two default schemas exist in every database automatically: `PUBLIC` and `INFORMATION_SCHEMA`. If you don't specify a schema, Snowflake assumes `PUBLIC`.

Common real-world pattern — layering schemas by data maturity stage:
- `RAW` → untouched data straight from source systems
- `STAGING` → partly cleaned, in-progress data
- `ANALYTICS` → final, trusted data for dashboards/reports

---

## Session Context — `USE` Statements
Snowflake maintains a **current session context**: the database/schema/warehouse/role you're "inside" at any moment.

```sql
USE DATABASE SALES_DB;
USE SCHEMA ANALYTICS;
USE WAREHOUSE etl_wh;

SELECT * FROM DAILY_SALES;  -- Snowflake infers SALES_DB.ANALYTICS.DAILY_SALES
```
In production pipelines/scripts, set context explicitly at the top rather than relying on defaults — avoids ambiguity bugs when multiple people/jobs share a session.

---

## Table Types — Critical for Interviews

| Type | Time Travel | Fail-safe | Use case |
|---|---|---|---|
| **Permanent** (default) | Up to 90 days (Enterprise+) | 7 days extra | Production data you must protect |
| **Transient** | Up to 1 day | None | Intermediate/staging data — cheaper storage, no long recovery needed |
| **Temporary** | 1 day, dies at session end | None | Session-scoped scratch data — disappears when session closes |

```sql
CREATE TRANSIENT TABLE SALES_DB.STAGING.ORDERS_STAGE (...);
CREATE TEMPORARY TABLE scratch_calc (...);
```

**Why it matters:** Time Travel and Fail-safe both cost storage. Using `TRANSIENT` for staging tables (truncated/reloaded daily anyway) saves real money at scale — a genuine cost-optimization decision engineers make, not academic trivia.

**Key terms:**
- **Time Travel** — ability to query/restore data as it existed at a past point in time (within the retention window).
- **Fail-safe** — an additional Snowflake-managed recovery period (7 days for permanent tables) after Time Travel expires, usable only by Snowflake support for disaster recovery — not self-service.

---

## Identifiers and Case Sensitivity (common bug source)
- Unquoted identifiers (`orders`, `ORDERS`, `Orders`) are automatically converted to **uppercase** internally. So `orders` and `ORDERS` refer to the same object.
- Wrapping an identifier in double quotes — `"orders"` — preserves case **exactly** and makes it case-sensitive. This causes real bugs when tools (BI tools, Python connectors) auto-quote identifiers and `"orders"` ends up ≠ `ORDERS`.

---

## Other Schema-Level Objects (beyond tables)
A schema is a general-purpose container, holding:
- **Views** — saved queries that behave like virtual tables
- **Stages** — staging areas for file loading
- **File Formats** — reusable CSV/JSON/Parquet parsing rule definitions
- **Sequences** — auto-incrementing number generators
- **Streams** — change-tracking objects (CDC — Change Data Capture)
- **Tasks** — scheduled SQL execution (like a cron job inside Snowflake)
- **Stored Procedures / UDFs (User-Defined Functions)** — custom logic

Each of these is covered in depth in upcoming topics.

---

## Practical / Project-Level Example (production-grade)

```sql
-- Production database and layered schemas
CREATE DATABASE SALES_DB;
CREATE SCHEMA SALES_DB.RAW;
CREATE TRANSIENT SCHEMA SALES_DB.STAGING;   -- entire schema marked transient: all tables inside inherit this
CREATE SCHEMA SALES_DB.ANALYTICS;

-- Raw landing table: permanent, since source data should be protected/recoverable
CREATE TABLE SALES_DB.RAW.ORDERS_RAW (
    order_id STRING,
    customer_id STRING,
    order_date STRING,
    amount STRING
);

-- Staging table: transient, since it's reloaded daily and doesn't need Time Travel/Fail-safe
CREATE TABLE SALES_DB.STAGING.ORDERS_CLEAN (
    order_id STRING,
    customer_id STRING,
    order_date DATE,
    amount NUMBER(10,2)
);

-- Set session context so downstream scripts don't need full paths repeatedly
USE DATABASE SALES_DB;
USE SCHEMA ANALYTICS;

-- Audit: check every table that exists in this database right now
SELECT TABLE_SCHEMA, TABLE_NAME, ROW_COUNT, BYTES 
FROM SALES_DB.INFORMATION_SCHEMA.TABLES;
```

This is a realistic production setup: layered schemas, deliberate table-type choices for cost control, explicit session context, and metadata auditing via `INFORMATION_SCHEMA`.

---

## Glossary — New Terms Introduced
| Term | Meaning |
|---|---|
| Namespace | A naming boundary that prevents naming conflicts between objects |
| INFORMATION_SCHEMA | Auto-generated metadata schema listing objects, columns, privileges, etc. |
| Session context | The current database/schema/warehouse/role active in your session (`USE` statements) |
| Permanent table | Default table type; full Time Travel + Fail-safe protection |
| Transient table | Cheaper table type; limited Time Travel, no Fail-safe — good for staging |
| Temporary table | Session-scoped table; disappears when session ends |
| Time Travel | Ability to query/restore past states of data within a retention window |
| Fail-safe | Snowflake-managed disaster recovery period after Time Travel expires (support-only) |
| Identifier case-sensitivity | Unquoted names become uppercase automatically; quoted names preserve exact case |
| Stream | Change-tracking object used for CDC (Change Data Capture) |
| Task | Scheduled SQL execution, like a cron job inside Snowflake |
| UDF | User-Defined Function — custom reusable logic |

---
**Next Topic:** Loading Data — Stages, COPY INTO, File Formats
