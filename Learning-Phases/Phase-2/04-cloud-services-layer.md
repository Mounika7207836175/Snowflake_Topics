# Phase 2 | Topic 4: Cloud Services Layer

## What This Layer Does
A collection of coordinated services managing the entire Snowflake account — not one single thing, but several distinct jobs:

1. **Authentication** — verifies identity at login (username/password, SSO, MFA).
2. **Infrastructure management** — provisions/manages virtual warehouses and storage behind the scenes.
3. **Metadata management** — tracks every table, its micro-partitions, value ranges (for pruning), row counts. Powers `INFORMATION_SCHEMA` and `SYSTEM$CLUSTERING_INFORMATION`.
4. **Query parsing and optimization** — reads submitted SQL, checks for errors, decides the most efficient execution approach (which partitions to scan, join order), generates an **execution plan** before handing it to a warehouse.
5. **Access control** — enforces RBAC (role-based access control), checking permissions before data is touched.

**Key detail:** this layer runs on Snowflake-managed compute, **separate** from virtual warehouses. Query parsing/planning happens *before* the warehouse starts working on data. Usually **not billed separately** (rare exception: "cloud services credits," only charged if this layer's usage exceeds 10% of daily compute spend — an edge case).

**Interview angle:** *"If your warehouse is suspended, can you still see table metadata (like row count)?"*
Yes — metadata is managed by the Cloud Services Layer, independent of any warehouse's active/suspended state. Actually running a query still needs an active/resumed warehouse.

## Hands-On: Seeing This Layer via Query Profile
In Snowsight: **Activity → Query History** → click a query → opens its **Query Profile**.

**Example — query:**
```sql
SELECT * FROM ORDERS_RAW WHERE amount > '200';
```

**Execution plan diagram (read bottom to top — actual execution order):**
1. **TableScan [2]** (runs first) — reads data from `SALES_DB.RAW.ORDERS_RAW`, pulling micro-partitions from storage.
2. **Filter [1]** (runs second) — applies the `WHERE amount > '200'` condition.
3. **Result [0]** (runs last) — returns final output columns.

- Circled numbers between boxes = **row count** passed from one step to the next (e.g., 3 rows passed TableScan → Filter → Result).
- Step IDs `[0]`, `[1]`, `[2]` are numbered in reverse of execution order.
- Percentages on each box = share of **total query time** spent in that step — useful for spotting bottlenecks on larger/slower queries.

**Statistics panel:** shows **Partitions scanned vs Partitions total** (ties to partition pruning — if scanned < total, pruning worked) and bytes scanned.

**On a small practice table:** Partitions scanned = 1, Partitions total = 1 (entire table fits in one micro-partition) — no pruning opportunity exists since there's only one partition. Pruning's value shows up on large tables with many partitions, where "scanned" would be meaningfully smaller than "total."

---
**Next Topic:** Separation of storage and compute
