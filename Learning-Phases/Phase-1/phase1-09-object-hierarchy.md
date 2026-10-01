# Phase 1 | Topic 9: Snowflake Object Hierarchy

## The Complete Hierarchy
```
Organization → Account → Database → Schema → Objects (Tables, Views, Stages, File Formats, Sequences, Streams, Tasks, Procedures, Functions)
```

## Level 1: Organization
Umbrella grouping multiple Snowflake accounts for centralized billing/management (see Topic 6).

## Level 2: Account
Your workspace — where data and compute physically live. Tied to one cloud provider and one region.

**Important exception:** **Warehouses, Users, and Roles are NOT part of the database hierarchy** — they exist directly at the **Account level**, outside the Database → Schema → Table chain.
- **Warehouse** = compute (processing power), not data.
- **User** = a person who logs in, not data.
- **Role** = a permission set, not data.

Only actual data-related objects live inside Database → Schema.

**Interview angle:** *"Is a virtual warehouse part of a database?"* No — it lives at the account level, independent of any database. One warehouse can query multiple databases.

## Level 3: Database
A container inside an account, grouping related data for one purpose/business area.

- **Namespace boundary:** two different databases can have identically-named schemas/tables with zero conflict, since Snowflake identifies objects by full path (`DATABASE.SCHEMA.TABLE`).
- Every database auto-generates an `INFORMATION_SCHEMA` (metadata: table sizes, row counts, columns, privileges).
```sql
SELECT * FROM SALES_DB.INFORMATION_SCHEMA.TABLES;
```
- A special system database `SNOWFLAKE` exists automatically in every account, holding account-wide usage metadata (query history, credit consumption, login history) — used for cost monitoring/auditing.
```sql
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
```
- Databases can be **cloned instantly** (zero-copy cloning — full topic later) without duplicating storage.

**Interview angle:** *"`SALES_DB.PUBLIC.CUSTOMERS` and `MARKETING_DB.PUBLIC.CUSTOMERS` — conflict?"* No — full path makes them distinct objects even with identical schema/table names.

## Level 4: Schema
Sits inside a database; organizes objects by purpose.

- Every database auto-creates two default schemas: `PUBLIC` (default landing spot if none specified) and `INFORMATION_SCHEMA`.
- **Security boundary:** access can be granted per schema (e.g., restrict an `HR` schema while `SALES` schema stays open), within the same database.
- Every schema also gets its own `INFORMATION_SCHEMA`.
- **Transient schemas:** all tables created inside automatically inherit transient behavior (limited Time Travel, no Fail-safe) without marking each table individually.
```sql
CREATE TRANSIENT SCHEMA SALES_DB.STAGING;
```
- **Managed Access Schemas:** the schema owner controls all grants to objects inside it, rather than each object's creator — centralizes permission management.
```sql
CREATE SCHEMA SALES_DB.FINANCE WITH MANAGED ACCESS;
```

**Interview angle:** *"Why a separate `HR` schema instead of one shared schema?"* Schemas are a security boundary — sensitive data can be isolated with stricter permissions while staying in the same database.

## Level 5: Objects Inside a Schema
| Object | What it is |
|---|---|
| **Table** | Stores actual rows/columns (Permanent/Transient/Temporary types) |
| **View** | Saved SQL query acting as a virtual table; stores no data, re-runs the query live on each access. A **Secure View** hides its underlying SQL logic even from users who can query it (useful for external sharing) |
| **Stage** | Waiting area for file loading (internal/external) |
| **File Format** | Reusable file-parsing rule definitions |
| **Sequence** | Auto-generates unique incrementing numbers (e.g., for IDs) — `CREATE SEQUENCE order_id_seq START = 1 INCREMENT = 1;` then `SELECT order_id_seq.NEXTVAL;` |
| **Stream** | Tracks inserts/updates/deletes on a table since it was last read — Snowflake's CDC (Change Data Capture) mechanism. Full topic later |
| **Task** | Scheduled SQL execution (like a cron job); often paired with Streams to build automated pipelines. Full topic later |
| **Stored Procedure** | Reusable procedural logic (loops, conditionals, multiple statements) in SQL/JavaScript/Python etc., called on its own |
| **UDF (User-Defined Function)** | Custom function used *inside* a query, must return a value usable in SQL |

**Interview angle:** *"Stored Procedure vs UDF?"* A UDF is used inside a query and returns a value directly usable in SQL. A Stored Procedure is called on its own, can perform multiple steps (including DDL like creating tables), and doesn't have to return a single SQL-usable value.

## Fully Qualified Names and Case Sensitivity
**Fully Qualified Name:** `DATABASE.SCHEMA.OBJECT_NAME` (e.g., `SALES_DB.ANALYTICS.DAILY_SALES`) — needed to avoid ambiguity, especially across different session contexts.

**Identifiers and case sensitivity**
- **Unquoted** identifiers (`orders`, `Orders`, `ORDERS`) are auto-converted to **uppercase** internally — all three refer to the same object.
- **Quoted** identifiers (`"orders"`) preserve exact case and become case-sensitive — `"orders"` ≠ `"ORDERS"`.
- **Real bug source:** tools that auto-quote identifiers (some BI tools, ORMs) can fail to find a table created unquoted, since `"Orders"` (quoted) ≠ `ORDERS` (stored uppercase from unquoted creation).

**Interview angle:** *"`CREATE TABLE Orders (...)` then `SELECT * FROM "Orders"` fails — why?"* Unquoted `Orders` is stored as `ORDERS`; the quoted `"Orders"` looks for that exact mixed case, which doesn't exist.

---
**Next Topic:** Fully qualified object names (focused follow-up)
