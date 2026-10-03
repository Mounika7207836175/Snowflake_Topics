# Phase 2 | Topic 8: Columnar Storage and Compression

## Row-Based vs Columnar Storage

**Row-based storage (traditional databases):** data is saved one entire row at a time, back-to-back:
```
[1001, C001, 2026-09-01, 250.00] [1002, C002, 2026-09-02, 175.50] ...
```
Good for fetching/updating a **whole record at once** (e.g., a single customer's order) — why transactional systems use it.

**Problem for analytics:** a query like `SUM(amount)` only needs the `amount` column, but row-based storage forces reading every row's full data (all columns) just to extract one column's values.

**Columnar storage (Snowflake's approach):** data is saved one entire column at a time:
```
order_id:    [1001, 1002, ...]
customer_id: [C001, C002, ...]
order_date:  [2026-09-01, 2026-09-02, ...]
amount:      [250.00, 175.50, ...]
```
For `SUM(amount)`, Snowflake reads **only** the `amount` column block, ignoring the rest entirely. For a 50-column table needing only 2 columns, this massively cuts the data physically read.

**Interview angle:** *"Why is columnar better for analytics but row-based better for transactional systems?"*
Analytical queries touch few columns but many rows — columnar reads only needed columns. Transactional systems need the whole row at once — row-based keeps that together efficiently.

## Why Columnar Storage Compresses Better

**Key insight:** values **within the same column** are far more similar to each other than values across a mixed row.
- `customer_id` column: same data type, often only a few hundred unique values repeating across millions of rows.
- A row mixes a number, a string, a date, a decimal — little repetition or pattern for a compression algorithm to exploit.

**Why similarity helps:** compression algorithms find and eliminate repeated patterns. Similar values sitting next to each other (as in one column) let the algorithm represent repetition very efficiently (e.g., storing a value once plus a repeat count, simplified).

**Real-world impact:** Snowflake often achieves **3x-5x+ compression ratios** from the columnar approach alone — reducing both storage cost and the amount of data that must travel from storage to compute during a query (faster queries, smaller bills, same root cause).

**Note on practice data:** compression benefits only show up clearly at scale (thousands/millions of rows) — a tiny practice table won't show a meaningful compression ratio, since there isn't enough repetition to compress against.

**Interview angle:** *"Why does columnar storage typically compress better than row-based?"*
Values within one column are far more similar/repetitive than values across a mixed-type row, so compression algorithms exploit that repetition more effectively.

---

## Side Topic: Is Snowflake OLAP-only? (OLTP vs OLAP, and Batch vs Streaming)

**OLTP (Online Transaction Processing):** systems for many small, fast read/write operations — one order placed, one row updated, one balance checked. Needs to handle thousands of tiny transactions/second, each touching few rows. Traditional row-based databases (MySQL, PostgreSQL, Azure SQL DB) excel here.

**OLAP (Online Analytical Processing):** systems for complex queries scanning huge volumes of data — aggregations, reporting, trend analysis, touching millions of rows but usually few columns. This is what Snowflake's columnar storage, micro-partitions, and MPP are built for.

**Why Snowflake isn't built for OLTP:**
- Columnar storage is inefficient for single-row lookups/updates — updating one row can involve rewriting an entire micro-partition behind the scenes.
- Virtual warehouses take time to resume (1-2+ seconds) — OLTP needs constant, instant availability for thousands of tiny transactions.
- Snowflake's pay-for-warehouse-time billing isn't cost-efficient for millions of tiny, constant transactions the way a dedicated OLTP database is.

**Nuance:** Snowflake's **Hybrid Tables** (part of **Unistore**) support some OLTP-style workloads (fast single-row lookups/updates) within the platform — a specialized addition, not Snowflake's primary design or typical job use. In practice, Snowflake handles OLAP and a separate system (e.g., PostgreSQL) handles OLTP.

**Correcting a common mix-up: OLTP/OLAP is NOT the same axis as Batch vs Streaming.**
- **OLTP vs OLAP** = about workload *type* (transaction size/frequency vs analytical complexity).
- **Batch vs Streaming** = about *when* data is processed (scheduled chunks vs continuous as it arrives) — a separate, independent dimension.

Snowflake supports **both** batch and near-real-time/streaming-style ingestion for OLAP workloads — covered later via **Snowpipe** (continuous loading) and **Streams** (tracking changes as they happen). It is OLAP-focused but not "batch only."

**Databricks is also OLAP, not OLTP** — it's a lakehouse-architecture platform for big data processing, data engineering, and machine learning, supporting both batch and streaming analytical workloads. It's a *competitor* to Snowflake in the data platform space, not an OLTP system.

**Corrected summary:**
| | OLTP | OLAP |
|---|---|---|
| Example systems | Azure SQL DB, PostgreSQL, MySQL | Snowflake, Databricks |
| Workload | Many small transactions | Complex analytical queries |
| Batch or streaming? | Neither — separate axis | Can be either — Snowflake supports both via Snowpipe/Streams |

**Interview angle:** *"Would you use Snowflake to power a live e-commerce checkout system?"*
No — that's an OLTP workload needing instant, high-frequency single-row transactions. Snowflake is built for OLAP: analyzing/reporting on that data afterward.

**Interview angle:** *"Is Databricks an OLTP or OLAP system?"*
OLAP — like Snowflake, built for analytical processing and big data workloads, not for powering live transactional applications.

---
**Next Topic:** Micro-partitions
