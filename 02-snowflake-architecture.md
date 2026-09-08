# Topic 2: Snowflake Architecture — Deep Dive

## The Three Layers (Recap)
Snowflake separates the system into 3 independent layers: **Storage**, **Compute (Virtual Warehouses)**, and **Cloud Services**.

---

## 1. Storage Layer — Where your data lives

Snowflake automatically splits your data into small chunks called **micro-partitions** (roughly 50–500MB each). Think of it as splitting one giant book into hundreds of small chapters.

For each micro-partition, Snowflake keeps a small "index card" noting what data range is inside (e.g., "this chapter only has March sales").

**Pruning**: When you run a query like `WHERE month = 'March'`, Snowflake checks the index cards first and *skips* chapters that don't match, instead of scanning everything. This is why Snowflake is fast on huge datasets without you manually creating indexes (unlike SQL Server, where you'd build indexes yourself).

You never manage micro-partitions directly — this is fully automatic.

---

## 2. Compute Layer — Virtual Warehouses ("the worker")

**Simple definition:** A Virtual Warehouse is a "worker" (or team of workers) that fetches and processes data to answer your queries, because raw data can't answer questions by itself.

### Warehouse Sizes ("how big is the worker-team")
`X-Small → Small → Medium → Large → X-Large → 2X-Large → ... → 6X-Large`

Each step up **doubles** the compute power — and doubles the cost per hour. Billing is measured in **credits** (1 credit ≈ 1 hour of one X-Small warehouse running).

### Auto-suspend / Auto-resume
- **Auto-suspend**: the worker "goes home" (stops) after N minutes of no activity, so you stop paying.
- **Auto-resume**: the worker automatically "comes back" the instant a new query arrives.

### Scale UP vs Scale OUT
- **Scale UP** = make your ONE worker-team bigger/stronger (e.g. Small → Large) → finishes **one big, heavy task** faster.
  - *Analogy:* One huge pizza order (1000 pizzas) for one event → add more chefs to the SAME kitchen.
- **Scale OUT** = add MORE separate identical worker-teams → serves **many people asking questions at the same time**, without anyone waiting in a queue. Does NOT speed up a single task.
  - *Analogy:* 50 different small orders from 50 customers at once → open more kitchen counters, each serving a different customer.

### Multi-cluster Warehouse
Instead of manually creating separate warehouses, you configure ONE warehouse to automatically "clone" itself into more identical teams when demand spikes, and shrink back down when it's quiet.

```sql
CREATE WAREHOUSE bi_wh 
  WAREHOUSE_SIZE = 'SMALL'        -- how big EACH team is
  MIN_CLUSTER_COUNT = 1           -- normally just 1 team
  MAX_CLUSTER_COUNT = 4           -- allow up to 4 teams during rush hour
  AUTO_SUSPEND = 300              -- go home after 300 seconds (5 min) idle
  AUTO_RESUME = TRUE;             -- come back automatically on new query
```
Real scenario: 50 analysts hit a dashboard at 9am → Snowflake auto-activates clusters 2, 3, 4. By 10am usage drops → it shrinks back to 1 cluster automatically. No manual intervention needed.

---

## 3. Cloud Services Layer — "the manager"

The invisible manager sitting above the workers. It doesn't do the actual querying, but it:
- Checks who's allowed to see what data (**security/access control**)
- Decides the smartest way to run a query before handing it to a warehouse (**query optimization**)
- Remembers where all the "index cards" (metadata) are

You don't create or separately pay for this layer in normal use — it's built into every Snowflake account.

---

## Practical / Project-Level Example

**Scenario:** You're the data engineer at an e-commerce company running two very different workloads.

```sql
-- Case 1: SCALE UP — one heavy nightly ETL job
-- ETL = Extract, Transform, Load: pulling raw data in, cleaning/reshaping it, then storing it properly
CREATE WAREHOUSE etl_wh 
  WAREHOUSE_SIZE = 'MEDIUM'    -- a stronger single team for one big task
  AUTO_SUSPEND = 60            
  AUTO_RESUME = TRUE;

-- Case 2: SCALE OUT — many analysts querying dashboards at once
-- BI = Business Intelligence: people using dashboards to make business decisions
CREATE WAREHOUSE bi_wh 
  WAREHOUSE_SIZE = 'SMALL'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 4
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE;
```

- `etl_wh` → one strong worker-team, for one heavy job that runs once a night.
- `bi_wh` → multiple identical worker-teams, spun up/down automatically to handle many people at once without anyone waiting.

This exact "scale up vs scale out" decision is a common real interview question for data engineers.

---

## New Terms Introduced (glossary)
| Term | Meaning |
|---|---|
| Micro-partition | A small automatic chunk (50-500MB) that your data is split into |
| Pruning | Skipping irrelevant micro-partitions during a query, based on stored metadata |
| Credit | Snowflake's billing unit — roughly 1 hour of one X-Small warehouse |
| Multi-cluster warehouse | One warehouse configured to auto-add/remove identical worker-teams based on demand |
| Cloud Services Layer | The "manager" layer — handles security, query optimization, metadata |
| ETL | Extract, Transform, Load — the process of pulling in raw data, cleaning it, and storing it properly |
| BI | Business Intelligence — using dashboards/reports to make business decisions |

## Self-Check (answered correctly by Mounika)
1. Virtual Warehouse = the "worker" that fetches/processes data to answer queries, since data can't answer by itself.
2. One huge slow nightly report → **Scale UP** (bigger single worker-team).
3. 50 people querying dashboards at once → **Scale OUT** (more separate worker-teams, avoids queueing).

---
**Next Topic:** Databases, Schemas, and Tables — Snowflake's object hierarchy
