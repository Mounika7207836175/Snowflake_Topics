# Phase 2 | Topic 6: Massively Parallel Processing (MPP)

## The Problem MPP Solves
If a single server had to read/process a huge table (e.g., 100 million rows) row by row alone, it would take a long time — how older, single-server databases worked.

## MPP's Solution — Step by Step
1. **Cloud Services Layer** decides which micro-partitions need scanning (from the execution plan).
2. Those micro-partitions are **divided among the available servers** in the warehouse (e.g., 400 partitions across 4 servers ≈ 100 each).
3. **Each server reads and processes its own slice independently, simultaneously** — no waiting on each other during this phase.
4. Each server calculates a **partial result** for its slice.
5. Partial results are **combined (aggregated)** into the final answer, returned to the user.

**Why it's faster:** work split across multiple servers running at the same time, roughly dividing total processing time (not perfectly linear due to coordination overhead, but the gain is real).

**Key clarification:** it's not the *query* that gets divided into slices — it's the **data (micro-partitions)** that gets divided among servers. Each server runs the *same* query logic against its *own* assigned chunk of data.

**Interview angle:** *"Why does increasing warehouse size often speed up large queries but not small ones?"*
MPP's speedup comes from splitting work across more servers. If there isn't enough data to meaningfully divide (a small table), extra servers sit idle — no speed gain, just extra cost.

## Data Skew — A Real Performance Problem
If one micro-partition (or a poorly distributed set) holds a disproportionate chunk of relevant data, one server can end up doing far more work than others. The whole query then waits on that one slow server, even if the rest finished quickly — sometimes called the **"long pole"** problem (overall time limited by the slowest piece, not the average).

**Interview angle:** *"3 out of 4 servers finish almost instantly, but the query still takes 30 seconds — what's happening?"*
Likely data skew — one server's slice of micro-partitions is disproportionately large/complex, so the query waits on that single straggling server even though parallelism worked for the others.

---
**Next Topic:** Virtual warehouses
