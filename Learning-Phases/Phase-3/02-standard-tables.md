# Phase 3 | Topic 2: Standard Tables

## Clarifying the Category
"Standard table" is an **umbrella category**, not a table type itself. Snowflake splits tables into two broad categories:

1. **Standard tables** — the traditional, native Snowflake table storage model. Contains **three sub-types**: Permanent, Transient, Temporary (next three topics).
2. **Specialty tables** — newer, purpose-built types: Dynamic, External, Apache Iceberg, Hybrid tables (later topics).

## What Makes a Table "Standard"
- Data stored **natively** in Snowflake's own managed storage, using its micro-partition format (Phase 2).
- Full support for **all** storage features: Time Travel, Fail-safe, zero-copy cloning, clustering keys, automatic micro-partitioning.
- This is the **default** table type — every `CREATE TABLE` run so far (`ORDERS_RAW`, `BIG_ORDERS`) has been a Standard table, specifically **Permanent** (the default sub-type when nothing else is specified).

**Specialty tables, for contrast (each gets its own topic):**
- **External tables** — point to data sitting *outside* Snowflake (e.g., files directly in Azure Blob, not loaded in).
- **Iceberg tables** — work with an open table format standard shared across platforms.
- **Hybrid tables** — support fast single-row OLTP-style operations (the Unistore feature from Phase 2's OLAP/OLTP discussion).
- **Dynamic tables** — auto-refresh based on a query definition.

Each trades away some Standard-table features in exchange for a specific capability.

**Interview angle:** *"Is 'Standard table' a specific table type like Permanent or Transient?"*
No — Standard is the umbrella category covering Permanent, Transient, and Temporary, all using Snowflake's native storage model. Distinguished from the newer Specialty types (External, Iceberg, Hybrid, Dynamic), which have different storage/behavior models.

## Hands-On: Confirming a Table's Actual Sub-Type

**What doesn't work (a real debugging lesson):** `SHOW TABLES IN SCHEMA ...` does NOT return a reliable `is_transient` column in its output — despite it seeming like a logical place to check.

**The reliable method — `GET_DDL`:** returns the exact `CREATE TABLE` statement Snowflake would use to recreate the object, showing the literal keyword used (or its absence).
```sql
SELECT GET_DDL('TABLE', 'SALES_DB.RAW.ORDERS_RAW');
```

**Result obtained:**
```sql
create or replace TABLE ORDERS_RAW (
	ORDER_ID VARCHAR(16777216),
	CUSTOMER_ID VARCHAR(16777216),
	ORDER_DATE VARCHAR(16777216),
	AMOUNT VARCHAR(16777216)
);
```
**No prefix keyword** (no `TRANSIENT`, no `TEMPORARY`) confirms `ORDERS_RAW` is a **Permanent table** — the default Standard sub-type.

**Practical takeaway:** when a documented/expected column or approach doesn't show up as expected, falling back to `GET_DDL` (or `DESCRIBE TABLE`) is a reliable way to verify an object's real definition directly, rather than guessing from `SHOW` command output.

---
**Next Topic:** Permanent tables
