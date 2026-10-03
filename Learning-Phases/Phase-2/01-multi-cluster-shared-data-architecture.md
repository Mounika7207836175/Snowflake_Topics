# Phase 2 | Topic 1: Multi-Cluster Shared-Data Architecture

## The Official Architecture Name
This is Snowflake's official architecture name — summarizing the separation of storage and compute covered earlier, with its proper term.

**"Shared-data"** — all compute (virtual warehouses) reads from **one single copy** of data in a central storage layer. Unlike older "shared-nothing" systems (each machine stores its own data slice), every Snowflake warehouse points to the same underlying storage. Data isn't duplicated or distributed per team — all compute clusters simply read the same source.

**"Multi-cluster"** — the compute side: multiple independent clusters (virtual warehouses) can run simultaneously, different sizes, serving different teams/workloads, without interfering with each other.

**Putting it together:** data lives in one shared place; compute exists as many independent, flexible clusters around it.

**Interview angle:** *"What is Snowflake's architecture called, and why?"*
Multi-cluster, shared-data architecture — "shared-data" because all compute reads from one central storage layer; "multi-cluster" because many independent compute clusters can run against that same data simultaneously.

## Hands-On Verification
```sql
-- Create a second warehouse
CREATE WAREHOUSE marketing_wh WITH WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;

-- Query a table with the original warehouse
USE WAREHOUSE COMPUTE_WH;
SELECT * FROM SALES_DB.RAW.ORDERS_RAW;

-- Query the SAME table with the new warehouse
USE WAREHOUSE marketing_wh;
SELECT * FROM SALES_DB.RAW.ORDERS_RAW;
```
**Result:** both warehouses return identical results — confirming there's only one copy of the data in storage, and each warehouse is just an independent "lens" reading it, able to run simultaneously without interference.

---
**Next Topic:** Storage layer
