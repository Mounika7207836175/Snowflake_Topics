# Phase 2 | Topic 3: Compute Layer

## What a Virtual Warehouse Actually Is
A virtual warehouse is a **cluster of compute resources** — one or more **servers** (nodes), each with its own CPU, memory, and local SSD disk for temporary/cached data. Warehouse size determines how many servers are bundled together.

**Server count by size (doubles each step up):**
| Size | Approx. Servers |
|---|---|
| X-Small | 1 |
| Small | 2 |
| Medium | 4 |
| Large | 8 |
| X-Large | 16 |
| 2X-Large | 32 |

## Why Doubling Size Doubles Cost
Credits (Snowflake's compute billing unit) are charged based on **how much raw compute power is used per hour**. Each size increase doubles the number of physical servers working, so it consumes exactly double the processing power — cost scales linearly and proportionally with the actual hardware used. No discount for bigger sizes.

**Practical implication:** a bigger warehouse only helps **data-heavy, computation-heavy queries**, where more servers can meaningfully split the work. A tiny query on a small table sees zero benefit from a bigger warehouse — there isn't enough data to parallelize, so you'd just pay more for the same speed.

**Interview angle:** *"If a simple query on a small table is slow, should you increase warehouse size?"*
No — size only helps when there's enough data/complexity to parallelize across more servers. For a small table, the bottleneck is usually something else (network latency, warehouse resuming from suspension), not lack of compute power.

## How Servers Process a Query
Servers in a warehouse split a query's work across themselves, each processing a different slice of data in parallel, then combine results. This is **Massively Parallel Processing (MPP)** — covered in depth in the next topic.

## Local Disk Caching
Each server has local SSD storage that **caches** recently-accessed micro-partitions. Re-running the same query shortly after (while the warehouse is still active) can read from this fast local cache instead of pulling from cloud storage again — noticeably faster. This cache is **lost** when the warehouse suspends or resizes.

**Interview angle:** *"Why might the same query run slower the first time than the second time?"*
First run reads from cloud storage (slower); second run can hit the warehouse's local SSD cache of recently-used micro-partitions (faster) — provided the warehouse hasn't suspended in between.

---
**Next Topic:** Cloud services layer
