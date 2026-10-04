# Phase 2 | Topic 17: How Snowflake Processes a Query (Capstone)

This topic sequences everything from Phase 2 into one continuous, end-to-end story — no new concepts, just the full journey of a single query.

## Example Query
```sql
SELECT customer_id, SUM(amount) AS total_spent
FROM SALES_DB.RAW.BIG_ORDERS
WHERE order_id BETWEEN 1000000 AND 2000000
GROUP BY customer_id
ORDER BY total_spent DESC;
```

## The Nine Stages

**Stage 1 — Submission and Authentication:** Cloud Services Layer authenticates the user and confirms the current role has `SELECT` privileges (RBAC check).

**Stage 2 — Result Cache Check:** Cloud Services checks if this exact query ran in the last 24 hours with no data changes, under the same role. If yes, everything below is skipped and the cached result returns immediately.

**Stage 3 — Parsing and Query Optimization:** SQL is parsed, checked for errors, and an execution plan is built — consulting the **Metadata Cache** for micro-partition statistics (min/max ranges) on `order_id`.

**Stage 4 — Partition Pruning Decision:** Using that metadata, Cloud Services determines which micro-partitions could possibly contain matching `order_id` values and marks the rest to skip.

**Stage 5 — Warehouse Assignment and Resume:** The execution plan is handed to the virtual warehouse (resuming if suspended). The Local Disk/Warehouse Cache is checked for any already-cached relevant partitions.

**Stage 6 — Massively Parallel Processing:** The warehouse's servers divide the surviving (non-pruned) micro-partitions among themselves, each scanning its slice in columnar format, decompressing as needed.

**Stage 7 — Local Computation and Aggregation:** Each server computes partial sums grouped by `customer_id` for its own slice — the `GROUP BY` executing in parallel.

**Stage 8 — Combining Results:** Partial results from all servers are combined into final totals, then sorted (`ORDER BY`).

**Stage 9 — Returning Results, and Caching for Next Time:** Final results return to the user, and Cloud Services stores the result in the Result Cache for potential reuse within 24 hours.

**The core principle tying Phase 2 together:** a query's speed is determined by how much work gets **avoided** at each stage — a Result Cache hit avoids everything; good pruning avoids scanning irrelevant partitions; good clustering (natural or explicit) is what makes pruning possible; MPP speeds up whatever work remains by splitting it across servers.

## Hands-On: Real Query Profile Results

Running the example query against `BIG_ORDERS` (36 total partitions) produced:

**Partitions scanned: 9 of 36.** Pruning worked — the range `1,000,000` to `2,000,000` spans more data than a single exact match (which pruned to 1 partition in Topic 11), so it naturally touches more partitions, but skipping 27 of 36 is still a meaningful win.

**Execution plan (bottom to top):**
1. **TableScan [4]** — reads `SALES_DB.RAW.BIG_ORDERS`, taking **90.9%** of total time — the dominant cost, as expected.
2. **Filter [3]** — applies `order_id >= 1000000 AND ...`.
3. **Aggregate [2]** — computes `SUM(amount)` — the `GROUP BY`.
4. **Sort [1]** — orders by `SUM(amount) DESC` — the `ORDER BY`.
5. **Result [0]** — final output: 500 rows (≈500 distinct customers in that `order_id` range).

**Profile Overview breakdown (new detail):**
- **Remote disk I/O: 22.7%** — time spent reading data from cloud storage.
- **Synchronization: 68.2%** — time spent coordinating between parallel servers (MPP overhead) — waiting for all servers to finish their slice before combining results. A real, measurable cost of parallelism, even on small practice data — directly connects to the "long pole"/data skew discussion from Topic 6.
- **Initialization: 9.1%** — compilation/planning step.

**Other stats:** `Bytes scanned: 4.63MB`, `Percentage scanned from cache: 0.00%` (Local Disk Cache didn't help here — this exact data hadn't been recently scanned by this warehouse in this form).

**Interview angle:** *"Besides actual data scanning, what else contributes to a parallel query's total execution time?"*
Synchronization overhead — the time spent coordinating between servers and waiting for all of them to finish their assigned slice before results can be combined. This overhead is real and measurable, not free, even though parallelism still provides a net speed benefit for large enough workloads.

---
**Phase 2 complete.**
