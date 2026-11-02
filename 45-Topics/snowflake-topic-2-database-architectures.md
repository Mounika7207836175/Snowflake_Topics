# Snowflake Notes: Topic 2, Database Architecture Types

An architecture answers one question: **how do the servers in a database share CPU, memory, and disk?** Scaling, speed, and cost all follow from that.

**Terms:** a **server** is one computer with its own CPU, memory, and disk. A **cluster** is a group of servers working together.

**Analogy (library with 1,000 books):**
- Shared everything: one librarian, one room, one shelf.
- Shared disk: 4 librarians, one common bookshelf.
- Shared nothing: 4 librarians, each owns their own shelf with 250 books.

---

## 1. Shared Everything (SMP)

**SMP** = Symmetric Multi-Processing. One big server. Many CPUs share **one memory and one disk**.

- *Example:* a single powerful Oracle or SQL Server machine running a company database.
- **Pros:** simple, no data movement between servers, easy to manage, good for small to medium workloads.
- **Cons:**
  - **Scaling:** only *scale up* (buy a bigger machine). Hard ceiling, steep cost.
  - **Contention:** all CPUs fight over the same memory and disk.
  - **Single point of failure.**
- **Bottleneck:** the shared memory and disk.

## 2. Shared Disk (Oracle RAC)

Multiple servers, each with its **own CPU and memory**, all connected to **one shared storage** (a SAN).

**Oracle RAC** (Real Application Clusters): 4 Oracle servers reading and writing the same disks.

- **Pros:**
  - Add servers for more compute without moving data.
  - If one server fails, others keep running (high availability).
  - No rebalancing, since every server sees all data.
- **Cons:**
  - **Contention:** servers edit the same data blocks, so they must coordinate ("cache fusion"). Under heavy writes this chatter slows everything.
  - The shared storage is specialized, costly hardware and becomes the bottleneck.
  - Adding more servers stops helping beyond a point.
- **Bottleneck:** the shared storage and server-to-server coordination.

## 3. Shared Nothing (MPP)

**MPP** = Massively Parallel Processing. Each server has its **own CPU, memory, and disk**. Servers talk only over the network. Examples: **Teradata**, **classic Redshift**.

Data is **distributed** across servers (for example by hashing `customer_id`). A query is split, each server scans its own slice in parallel, and results are combined.

- *Example:* 10 servers, 10 TB table, so each holds 1 TB. A scan runs on all 10 at once.
- **Pros:** scales *out* by adding servers, very fast big scans and aggregations, no shared bottleneck.
- **Cons:**
  - **Rebalancing:** adding or removing servers means data must be physically redistributed. Example: 4 servers with 8 TB (2 TB each). Add a 5th server, and about 1.6 TB per server must be redistributed. This takes hours or days and slows queries meanwhile.
  - **Storage and compute are tied together:** more storage means buying more compute too.
  - **Data skew:** a bad distribution key (60% of rows share one value) overloads one server, and the whole query waits for it.
  - **Concurrency:** everyone shares the same fixed servers, so heavy ETL slows dashboards.
  - Needs distribution keys, sort keys, and maintenance such as VACUUM.
- **Bottleneck:** the slowest server (skew), the network during shuffles, and the rebalancing event.

## 4. Hybrid: Snowflake

Snowflake takes the best of both:
- Like **shared disk:** one central copy of data in cloud storage that all compute can read.
- Like **shared nothing:** each virtual warehouse is an MPP cluster with its own CPU, memory, and local SSD cache, processing in parallel.

**Three layers**
1. **Storage:** one central copy in cloud object storage (Azure Blob/ADLS, S3, GCS).
2. **Compute:** independent **virtual warehouses**.
3. **Cloud services:** metadata, security, optimizer, transactions.

**What this fixes**
- **No rebalancing:** compute servers hold no permanent data, so resizing a warehouse moves nothing. (They keep only a temporary local cache.)
- **No contention between teams:** ETL, BI, and Data Science each get their own warehouse reading the same data.
- **Independent scaling:** storage and compute grow separately.

**Multi-cluster warehouse:** one warehouse can run several identical clusters (for example min 1, max 4). When many users query at once, Snowflake starts extra clusters automatically and shuts them down after.
- *Scale up* (bigger size) helps **one heavy query**.
- *Scale out* (more clusters) helps **many concurrent queries**.

**Shared data:** all warehouses, and even other accounts, read the same stored data. No copies, one source of truth.

### MPP, and where Snowflake fits

| | Classic MPP (shared nothing) | Snowflake |
|---|---|---|
| Parallel processing by many servers | Yes | Yes |
| Where data lives | On each server's own disk | In central storage |
| Adding servers | Must rebalance data | No data movement |

Snowflake is MPP for **processing** and shared-disk-like for **storage**, which is why it is called a hybrid.

---

## 5. Comparison

| | Shared Everything | Shared Disk | Shared Nothing (MPP) | Snowflake (Hybrid) |
|---|---|---|---|---|
| Example | Single SQL Server | Oracle RAC | Teradata, classic Redshift | Snowflake |
| CPU/Memory | Shared | Separate per server | Separate per server | Separate per warehouse |
| Disk | Shared | Shared | Separate per server | Shared central storage |
| Scales | Up only | Out (limited) | Out (with rebalancing) | Out and up, independently |
| Main bottleneck | Memory + disk | Shared storage + coordination | Slowest server, shuffles, rebalance | Usually query design or warehouse size |
| Concurrency | Poor | Medium | Medium | Strong |

---

## 6. Why Snowflake does not have the shared-disk problem

Snowflake also has many servers reading one storage, so why is it fine?

1. **Different storage.** Oracle RAC uses one specialized SAN with limited speed. Snowflake uses cloud object storage, spread across many machines inside the cloud, so it handles many readers at once.
2. **Immutable micro-partitions.** In Oracle RAC, servers edit the same data blocks in place and must coordinate. Snowflake stores data in **micro-partitions that are never edited in place**. An UPDATE writes a **new** micro-partition and updates the metadata to point to it. Queries already running keep reading the old one and finish safely. Nothing to fight over.
3. **Local cache.** Each server keeps recently used data on local SSD, reducing trips to central storage.
4. **Separate warehouses per team.** Heavy work in one warehouse does not slow another.

**Memory aid:** shared disk shares data that gets edited. Snowflake shares data that is never edited in place.

### Pruning vs immutability (do not mix these up)

| Idea | What it gives you |
|---|---|
| **Pruning** (micro-partition metadata, such as min/max values per column) | **Speed:** reads only the micro-partitions that could hold your data |
| **Immutable micro-partitions** | **No blocking:** readers never clash with changes |

Side benefit of immutability: old micro-partitions are kept for a while, which is what makes **Time Travel** possible (later topic).

---

## 7. OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Meaning | Online Transaction Processing | Online Analytical Processing |
| Purpose | Run the business day to day | Analyze the business |
| Operations | Many small INSERT/UPDATE/DELETE | Few huge reads and aggregations |
| Rows touched | A few (one order) | Millions (all orders this year) |
| Design | Row-oriented, normalized | Column-oriented, denormalized |
| Users | Customers, apps | Analysts, managers, BI |
| Example | Banking app, order system (MySQL, Postgres) | Snowflake, Redshift, BigQuery |

*Scenario:* placing an online order is recorded by an **OLTP** database in milliseconds. At night the data is copied into **OLAP** (Snowflake), where management asks "total sales by region this quarter". Running that heavy question on the OLTP database would slow the live shop, so they are kept separate.

Note: streaming vs batch describes **how data arrives**, which is a different idea from OLTP vs OLAP.

---

## 8. Data lake vs data warehouse vs lakehouse

| | Data Warehouse | Data Lake | Lakehouse |
|---|---|---|---|
| Data type | Mainly structured, cleaned | **All types:** structured, semi-structured, unstructured (files, JSON, images, logs) | All types |
| Schema | Defined **before** loading (schema-on-write) | Applied **when reading** (schema-on-read) | Flexible, with enforced tables on top |
| Cost | Higher storage cost | Very cheap storage | Cheap storage |
| Users | Analysts, BI | Data scientists, engineers | All |
| Weakness | Rigid, costly for raw data | Can become a messy "data swamp", no transactions | Newer, still maturing |
| Example | Teradata, classic Redshift | Files in Azure Data Lake Storage | Databricks (Delta Lake), Snowflake with Iceberg tables |

*Scenario:* raw clickstream JSON sits in a **lake** (cheap, flexible). Cleaned sales tables sit in the **warehouse** (fast, trusted reports). A **lakehouse** aims to give both in one system.

Snowflake spans all three ideas: warehouse features, semi-structured storage like a lake, and lakehouse-style open table formats.

---

## Quick recap
1. Shared everything: one server, scales up only.
2. Shared disk: many servers, one storage, coordination bottleneck.
3. Shared nothing: servers with their own storage, rebalancing when servers are added, storage tied to compute.
4. Snowflake: separate compute servers + central storage + immutable micro-partitions, so no rebalancing and no blocking.
5. OLTP runs the business; OLAP analyzes it.
6. Lake = all data types, raw and cheap; warehouse = structured and fast; lakehouse = both.

---

## Hands-on: draw the three architectures

Draw three boxes side by side and mark the bottleneck in each:

1. **Shared everything:** one big box with CPUs, one memory block, one disk. Mark the bottleneck on the memory and disk.
2. **Shared disk:** several servers (CPU + memory each), all with arrows into one storage cylinder. Mark the bottleneck on the storage and on the coordination between servers.
3. **Shared nothing:** several servers, each with its own CPU, memory, and disk. Mark the bottleneck on the slowest/overloaded server and on the network arrows (shuffles, rebalancing).

**Bonus (4th box, Snowflake):** central storage at the bottom, three warehouses above it (ETL, BI, Data Science), and the cloud services layer on top. No shared bottleneck between warehouses.
