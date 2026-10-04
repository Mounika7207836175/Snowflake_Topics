# Phase 2 | Topic 15: Warehouse Cache

Same concept as **Local Disk Cache**, covered in depth in the Virtual Warehouses topic — this is simply the name used for it here.

## Recap
Each server in a warehouse has local SSD storage that caches recently-scanned micro-partitions. Re-running a query that still needs to touch storage (i.e., not an exact Result Cache match) can read from this cache instead of pulling from cloud storage again — provided the warehouse hasn't suspended or resized since.

| | Detail |
|---|---|
| Lives in | Inside the warehouse, on each server's local SSD |
| Stores | Recently-scanned raw micro-partition data |
| Requires active warehouse? | Yes |
| Lost when | Warehouse suspends or resizes |

## New Detail: Multi-Cluster Warehouses Don't Share Cache
If a warehouse is a **multi-cluster** warehouse (Enterprise edition and above), each individual cluster maintains its **own separate** local disk cache — clusters do **not** share this cache with each other.

**Practical implication:** if Cluster 1 scans certain micro-partitions and Cluster 2 later gets a query needing the same data, Cluster 2 gets **no benefit** from Cluster 1's cache — it reads fresh from storage, just as if no caching had happened at all.

**Interview angle:** *"In a multi-cluster warehouse, does one cluster's cache help another cluster's query?"*
No — each cluster maintains its own independent local disk cache; there's no cache sharing between clusters, even within the same warehouse.

---
**Next Topic:** Metadata cache
