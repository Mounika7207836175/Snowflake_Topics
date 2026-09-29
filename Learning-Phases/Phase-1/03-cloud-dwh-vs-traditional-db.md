# Phase 1 | Topic 3: Cloud Data Warehouse vs Traditional Database

## 1. Hosted-in-Cloud vs Cloud-Native
- **Hosted in the cloud:** e.g., creating an Azure VM and installing SQL Server on it. Same old limitations remain — manual capacity planning, storage+compute bundled, manual maintenance, hard data sharing. This is "a database running on a cloud server," not a cloud data warehouse.
- **Cloud-native:** the warehouse/platform is built to run on the cloud from the ground up (e.g., Snowflake). It doesn't carry those old limitations.

**Note on Snowflake's identity:** Snowflake is not a traditional relational database, and "data warehouse" undersells it too (it also covers data lakes, pipelines, sharing, apps). The accurate term is **cloud-native data platform** — "warehouse" is often used loosely since that's still its core use case.

## 2. Side-by-Side Comparison (Azure SQL Server on a VM vs Snowflake)

| Aspect | Traditional (e.g., SQL Server on Azure VM) | Snowflake (cloud-native) |
|---|---|---|
| Storage & Compute | Bundled on the same VM | Fully separate — storage in cloud blob storage, compute as independent virtual warehouses |
| Scaling | Manual — resize VM, often needs downtime | Elastic — resize/add compute in seconds, no downtime |
| Maintenance | You patch OS/SQL Server, manage indexes, backups | Automated by Snowflake |
| Concurrency | One shared engine; heavy queries slow others | Multiple independent warehouses on same data; no contention |
| Semi-structured data | Needs extra tools/columns for JSON | Native VARIANT type |
| Data sharing | Export files, manual, stale copies | Live, secure sharing, no copies |
| Pricing | Pay for the VM whether used or not | Pay separately: storage (flat) + compute (only while running) |

## Interview One-Liner
"A traditional database might run *in* the cloud, but a cloud-native platform like Snowflake was *built for* the cloud — separated storage/compute, automated maintenance, and elastic scaling are the real differentiators, not just where the server physically sits."

---
**Next Topic:** Snowflake editions
