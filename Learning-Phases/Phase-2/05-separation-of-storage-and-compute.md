# Phase 2 | Topic 5: Separation of Storage and Compute

## The Core Idea
Storage Layer (micro-partitions in cloud blob storage) and Compute Layer (virtual warehouses) are two **independently scalable, independently billed** components, coordinated by the Cloud Services Layer.

## Billing Independence (in exact detail)
- **Storage billing:** charged purely on **volume stored** (TB/month), regardless of how many warehouses exist or how often data is queried.
- **Compute billing:** charged purely on **credits consumed while a warehouse is actively running**, regardless of how much data is stored. A 1TB table and a 1GB table cost the same to query with the same warehouse for the same duration (though a bigger table may need more compute time to scan, indirectly affecting cost).

**Why this matters:** deleting/suspending every warehouse in an account leaves data completely safe and intact in storage (only the small storage fee continues). A new warehouse can be created anytime to query it again — storage and compute don't depend on each other's existence.

**Interview angle:** *"If you suspend/drop all virtual warehouses, what happens to your data?"*
Nothing — it stays fully intact in storage. Only the ability to *query* it is lost until a warehouse is created/resumed.

## How the Three Layers Actually Interact (corrected flow)
The three layers (Cloud Services, Compute, Storage) are stacked and independent — not a simple A→B→C pipeline where data flows through all three in sequence.

**Actual query flow:**
1. Query submitted → goes to **Cloud Services Layer** first.
2. Cloud Services parses it, checks permissions (security), checks its **metadata** (which micro-partitions exist, their value ranges) to build an execution plan — deciding which partitions need scanning.
3. Cloud Services hands the plan to a **Virtual Warehouse** (Compute) — telling it what to run and which partitions to use.
4. The **Virtual Warehouse reads those micro-partitions directly from Storage** — compute pulls data bytes straight from storage. Cloud Services does **not** sit in the middle relaying the actual data.
5. The warehouse processes the data and returns results.

**Key correction:** Cloud Services doesn't act as a physical data pipe between Compute and Storage. It **coordinates** both — directing compute on what/where via metadata and the execution plan — while actual data retrieval happens directly between Compute and Storage.

**Interview angle:** *"Does data physically pass through the Cloud Services Layer when a warehouse reads from storage?"*
No — Cloud Services directs and plans (via metadata/execution plan); actual data transfer happens directly between Compute and Storage.

---
**Next Topic:** Massively Parallel Processing (MPP)
