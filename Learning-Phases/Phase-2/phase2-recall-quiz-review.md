# Phase 2 Recall Quiz — Answers Review

## Topics 1-2: Architecture / Storage Layer

**Q1. What does "multi-cluster, shared-data" mean, precisely?**
Her answer: Data is stored centrally in the storage layer; compute layer has multiple independent virtual warehouses (sized at creation), each computing results on the same data.
✅ Correct.

**Q2. Four steps data goes through, from loaded to micro-partition?**
Her answer: Collected from sources → compressed → stored in columnar format → stored as micro-partitions, placed in cloud provider storage (Azure Blob/S3).
✅ Correct.

## Topics 3-5: Compute / Cloud Services / Separation

**Q3. Why does doubling warehouse size double its cost?**
Her answer: A bigger warehouse does more work, and each warehouse has its own CPU/local disk, so it costs based on usage.
⚠️ Right spirit, imprecise mechanism. Precise answer: doubling size doubles the number of physical servers = doubles the compute power consumed = Snowflake charges proportionally/linearly for that extra hardware.

**Q4. Why doesn't a bigger warehouse speed up a query on a tiny table?**
Her answer: A warehouse is a collection of servers; a tiny table (e.g., 5 rows) doesn't need multiple servers — one server can read it directly. A huge table needs the data split across 4-5 servers for parallel processing.
✅ Correct.

**Q5. Name three distinct jobs the Cloud Services Layer does.**
Her answer: Improves query performance, infrastructure management, stores metadata (roles, table value ranges).
⚠️ Partial. "Improves query performance" isn't the precise job — the actual job is **parsing and generating the execution plan** (performance improvement is a result of this, not the job itself). Infrastructure management and metadata management are correct. Two more genuine jobs: **authentication** and **access control (RBAC)**.

**Q6. Does a warehouse reading from storage pass through the Cloud Services Layer?**
Her answer: No — Cloud Services sits on top, stores metadata and designs the execution plan, but doesn't sit in the middle; the warehouse reads directly from storage.
✅ Correct.

**Q7. Why can you delete every warehouse without losing data?**
Her answer: Storage and compute are independent (unlike traditional databases where they're bundled) — data sits in the storage layer, warehouses sit in the compute layer, so deleting warehouses doesn't touch data.
✅ Correct.

## Topic 6: MPP

**Q8. Is it the query or the data that gets divided among servers in MPP?**
Her answer: The data gets divided, not the query — each server runs the *same* query but on a different range/slice of the data (e.g., 10,000 rows split as 5,000 + 5,000 across 2 servers).
✅ Correct.

**Q9. What is data skew, and why can it slow a query even if most servers finish quickly?**
Her answer: Described it as "data locked inside a warehouse" and was unsure of the mechanism (flagged herself as weak here).
⚠️ Needs correction. Data skew = one server's assigned slice of micro-partitions is disproportionately large/complex, so that server takes much longer than the others. The whole query waits on that single straggling server (the "long pole" problem) even though the rest finished fast — it has nothing to do with data being "locked."

## Topic 7: Virtual Warehouses

**Q10. Difference between Local Disk Cache and Result Cache, in terms of where each lives?**
Her answer: Local Disk Cache lives inside the virtual warehouse (needs an active/started warehouse; lost on suspend). Result Cache lives in the Cloud Services Layer, stores final query results, persists 24 hours.
✅ Correct.

**Q11. If you resize a warehouse while a query is running, what happens to that query?**
Her answer: The running query doesn't stop — it continues with the old configuration; the resize applies to the next queries.
✅ Correct.

## Topic 8: Columnar Storage

**Q12. Why is columnar storage better for `SUM(amount)`-style aggregation?**
Her answer: Columnar storage groups a column's data together, so aggregation only needs to read that one column instead of the whole table/different data types — better for OLAP-style reports.
✅ Correct.

**Q13. Why does columnar storage compress better than row-based?**
Her answer: Compression finds similar patterns; a column holds one data type (e.g., all `customer_id` values), which compresses well. A row mixes different data types (ID, name, etc.), making it rare to find similar patterns, so it compresses poorly.
✅ Correct.

**Q14. Is Snowflake built for OLTP or OLAP? Why?**
Her answer: OLAP, because of columnar storage; mentioned a newer feature (name not recalled) letting Snowflake support some OLTP too.
✅ Correct (the feature is **Hybrid Tables / Unistore**).

## Topic 9: Micro-Partitions

**Q15. If you update one row, does Snowflake modify it in place?**
Her answer: No — it creates a new partition with the new data instead of modifying in place.
✅ Correct.

## Topic 10: Micro-Partition Metadata

**Q16. What specific metadata does Snowflake store for each micro-partition?**
Her answer: Range of values, total number of partitions, average depth, and bytes.
⚠️ Mixed two levels together. **Per-partition** metadata = range of values per column, distinct value counts, row count, partition size (bytes). `total_partition_count` and `average_depth` are **table-level aggregate stats** reported by `SYSTEM$CLUSTERING_INFORMATION` — not something stored individually per partition.

## Topic 11: Partition Pruning

**Q17. What does pruning mean, and what does it depend on?**
Her answer: The process of reading fewer partitions than the total, by skipping partitions that can't contain relevant data (e.g., 1 out of 35). Possible because of clustering (natural or explicit keys).
✅ Correct.

## Topic 12: Natural Data Clustering

**Q18. Why does natural clustering degrade over time, even with no structural changes?**
Her answer: Said it's "the output of what a user types" and doesn't always update/cluster the data — unclear on the actual mechanism.
⚠️ Needs correction. Micro-partitions are immutable — every update/insert creates new partitions appended at the "end" of the partition list, not reinserted into sorted position. This causes increasing overlap with the original partitions over time, degrading clustering, even without any change to the table's structure.

**Q19. Does Snowflake "choose" a column to naturally cluster by?**
Her answer: No — it depends on the order data is inserted in, not a deliberate choice by Snowflake.
✅ Correct.

## Topic 13: Clustering Keys

**Q20. Real difference between natural clustering and an explicit clustering key?**
Her answer: With an explicit key, the process is continuous — Snowflake checks over time and creates new partitions as needed. Natural clustering only creates new partitions as a result of the user's own actions (update/insert), not continuously.
✅ Correct — precisely the right distinction (who initiates the work, and how often).

**Q21. Does automatic reclustering affect Time Travel?**
Her answer: No — old partitions remain available even with automatic reclustering.
✅ Correct.

## Topics 14-16: The Three Caches

**Q22. Four conditions for a Result Cache hit?**
Her answer: Same query (spacing ignored, but case sensitivity/line-by-line may matter), within 24 hours, no data change since execution.
⚠️ 3 of 4 given. Missing: the **same role/access rights** must apply — a different role that would see different data (e.g., due to row-level security) won't reuse another role's cached result.

**Q23. Why can you query INFORMATION_SCHEMA with zero active warehouses, but not an actual table?**
Her answer: INFORMATION_SCHEMA sits in the Cloud Services Layer, not the storage layer — no active warehouse/compute needed to fetch it.
✅ Correct.

## Topic 17: Query Processing

**Q24. Besides actual data scanning, what else can take significant time in a parallel query's execution?**
Her answer: Said parallel processing improves performance by splitting data across servers, but didn't identify the specific cost — flagged not understanding the question.
⚠️ Not answered. Correct answer: **Synchronization overhead** — time spent coordinating between parallel servers and waiting for all of them to finish their assigned slice before results can be combined. This was directly observed in her own Topic 17 hands-on Query Profile (68.2% synchronization time).

---

## Summary — Areas to Revisit
- Data skew mechanism (Q9)
- Cloud Services Layer's precise jobs — parsing/execution plan vs "improving performance" (Q5)
- Per-partition metadata vs table-level aggregate stats — the distinction between the two (Q16)
- Why natural clustering degrades — the immutability/append mechanism (Q18)
- Fourth Result Cache condition — matching role/access rights (Q22)
- Synchronization overhead as a real cost in parallel query execution (Q24)
