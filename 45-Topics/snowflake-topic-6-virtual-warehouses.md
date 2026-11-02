# Snowflake Notes: Topic 6, Virtual Warehouses

A virtual warehouse is a cluster of servers that runs queries and stores no permanent data. This topic is about **choosing and tuning it**.

## 1. Sizes (X-Small to 6X-Large)

| Size | Servers | Credits/hour |
|---|---|---|
| X-Small | 1 | 1 |
| Small | 2 | 2 |
| Medium | 4 | 4 |
| Large | 8 | 8 |
| X-Large | 16 | 16 |
| 2X-Large | 32 | 32 |
| 3X-Large | 64 | 64 |
| 4X-Large | 128 | 128 |
| 5X-Large | 256 | 256 |
| 6X-Large | 512 | 512 |

Each size doubles the servers and the cost. The two biggest sizes may not be available in every cloud region. You will rarely need more than X-Large or 2X-Large in normal work.

**Credits per hour vs credits used:** the table shows credits per hour. Credits used = running time in hours x credits per hour.

*Example:* a Medium warehouse (4 credits/hour) running for 30 minutes uses 0.5 x 4 = **2 credits**. An X-Small running the same 30 minutes uses 0.5 x 1 = **0.5 credits**.

## 2. Scale up vs scale out

| | Scale **up** | Scale **out** |
|---|---|---|
| What changes | Bigger size (more servers per cluster) | More clusters of the same size |
| Helps with | One **heavy, slow query** | **Many users** at the same time (queuing) |
| How | `ALTER WAREHOUSE ... SET WAREHOUSE_SIZE = 'LARGE'` | Multi-cluster warehouse |

*Scenario:* one monthly report takes 30 minutes: scale **up**. 200 analysts open dashboards at 9 AM and queries wait in a queue: scale **out**.

## 3. Auto-suspend and auto-resume

- **AUTO_SUSPEND = n seconds:** shuts the warehouse after n idle seconds, and billing stops. Setting it to 0 or NULL means never suspend, which is expensive.
- **AUTO_RESUME = TRUE:** starts it again when a query arrives, usually within a couple of seconds.
- Billing is per second, with a **60-second minimum** each time it starts.
- **Trade-off:** when a warehouse suspends, its **local cache is lost**. A very short auto-suspend saves credits but makes repeat queries slower.
- Rule of thumb: about 60 seconds for ETL and ad-hoc work, longer (5 to 10 minutes) for dashboards that people reopen often, so the cache stays warm.

## 4. Multi-cluster warehouses (Enterprise edition and above)

One warehouse that can run **several clusters** at once.
- `MIN_CLUSTER_COUNT` and `MAX_CLUSTER_COUNT` (up to 10).
- **Auto-scale mode** (min < max): Snowflake starts extra clusters when queries queue, and removes them when demand drops. Example: min 1, max 4.
- **Maximized mode** (min = max, more than 1): all clusters always run.
- Credits = size credits x number of **running** clusters. A Medium warehouse with 3 running clusters costs 12 credits/hour.

```sql
CREATE WAREHOUSE bi_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 4
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300;
```

## 5. Scaling policy: Standard vs Economy

Only matters for multi-cluster warehouses.

**Picture a supermarket with billing counters.** Each counter is a cluster. When the queue grows, the manager can open another counter, but each open counter costs money.

| | Standard | Economy |
|---|---|---|
| Goal | Avoid queuing (performance first) | Save credits |
| Adds a cluster | Soon after queries start to queue | Only when there is enough work to keep a new cluster busy for about 6 minutes |
| Shuts a cluster down | After a few minutes of low load | After a longer period of low load |
| Trade-off | Higher cost, rarely any wait | Lower cost, queries may wait |

*Example:* a warehouse allows 1 to 4 clusters. At 9 AM, 100 dashboard queries arrive. With **Standard**, clusters 2, 3, and 4 start quickly and everyone gets fast answers (higher cost). With **Economy**, Snowflake holds back and lets some queries wait, starting extra clusters only if the load really lasts (lower cost).

Use Standard for dashboards, where people don't want to wait. Use Economy for batch jobs and reports, where waiting a minute is fine.

## 6. Query Acceleration Service (QAS)

**Picture:** one chef (your warehouse) handles a normal day of orders. One day, a single huge order arrives: peel 1,000 kilos of potatoes. QAS is like **hiring temporary helpers just for the peeling part** of that one order. They leave when it's done, and you pay only for the time they worked.

In Snowflake terms:
- Your warehouse is sized for normal queries (say, Small).
- Once in a while, a query scans a **huge amount of data** (for example 5 TB, filtering for just a few rows).
- With QAS on, Snowflake sends part of that query (the heavy scanning) to extra Snowflake-managed compute. Your warehouse handles the rest.
- You are billed separately for that extra compute, only when it is used.

**Why not just make the warehouse bigger?** A bigger warehouse costs more for **every** query, even small ones. QAS adds power only for the rare big query.

- Needs Enterprise edition or higher.
- Helps mostly with big scans and filters. It does not help every query.
- If **all** your queries are heavy, use a bigger warehouse instead.

```sql
ALTER WAREHOUSE my_wh SET
  ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;

-- check whether a query would benefit
SELECT SYSTEM$ESTIMATE_QUERY_ACCELERATION('<query_id>');
```

*Scenario:* a team runs normal dashboards all day, but once a week an analyst runs a query that scans 3 TB. QAS handles that one query without paying for a bigger warehouse all week.

## 7. Snowpark-optimized warehouses

**Snowpark** is a way to write **Python (or Java or Scala) code that runs inside Snowflake**, instead of only SQL. The data stays in Snowflake, and your code runs next to it.

Some Python work needs a **lot of memory** on each server, for example training a machine learning model on tens of millions of rows. A normal warehouse has standard memory per server, so such a job can run out of memory or slow down by spilling to disk.

A **Snowpark-optimized warehouse** has servers with **much more memory each**.

**Picture:** a normal car vs a truck. The car (standard warehouse) is perfect for everyday trips (SQL queries). The truck (Snowpark-optimized) carries heavy loads (memory-hungry Python and ML jobs) but costs more to run.

```sql
CREATE WAREHOUSE ml_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  WAREHOUSE_TYPE = 'SNOWPARK-OPTIMIZED';
```

- Smallest size is Medium.
- Costs more credits per hour than a standard warehouse of the same size (check Snowflake's current pricing table).
- Don't use it for normal SQL. A standard warehouse is cheaper and fine for that.

*Example:* a data scientist trains a model in Python on 50 million rows. On a standard warehouse it fails with an out-of-memory error. On a Snowpark-optimized warehouse the same job completes.

## 8. Sizing for cost

**Key idea:** if a query scales well, **doubling the size halves the time and costs the same**.

*Example:* a query takes 8 minutes on X-Small (1 credit/hour).
- X-Small: 8/60 hour x 1 = **0.133 credits**
- Medium (4 servers), about 2 minutes: 2/60 hour x 4 = **0.133 credits**

Same cost, 4x faster. So often **bigger is not more expensive**.

**It stops working when:**
- The query is small (a few seconds), so a bigger size just wastes credits (and the 60-second minimum applies).
- The query can't use more servers (a tiny table, or a step that runs on a single server).

**Practical rules**
1. **Separate warehouses per workload** (ETL, BI, data science), each sized for its own work.
2. Start small, and go up only when queries are slow or the Query Profile shows **spilling to disk** (data too big for memory), a sign the warehouse is too small.
3. Use **multi-cluster** for concurrency (queuing), a **bigger size** for slow single queries.
4. Keep auto-suspend short where cache doesn't matter.
5. Set **resource monitors** to alert or suspend when credits hit a limit.

---

## Quick recap
1. Sizes double servers and credits per hour each step, from 1 to 512 servers.
2. Scale up = heavy single query. Scale out = many users.
3. Auto-suspend stops billing (60-second minimum per start) but clears the cache.
4. Multi-cluster needs Enterprise. Standard policy avoids queuing, Economy saves credits.
5. QAS = temporary extra helpers for rare giant queries. Snowpark-optimized = memory-heavy warehouse for Python and ML jobs.
6. Well-scaling queries cost about the same on a bigger size, so often faster is free.

---

## Hands-on: same query on X-Small and Medium, compare time and credits (do later on a laptop)

**Step 1: create two warehouses**
```sql
CREATE WAREHOUSE wh_xs WAREHOUSE_SIZE='XSMALL' AUTO_SUSPEND=60 AUTO_RESUME=TRUE INITIALLY_SUSPENDED=TRUE;
CREATE WAREHOUSE wh_m  WAREHOUSE_SIZE='MEDIUM' AUTO_SUSPEND=60 AUTO_RESUME=TRUE INITIALLY_SUSPENDED=TRUE;
```

**Step 2: make the test fair.** Turn off the result cache so Snowflake doesn't just return the saved answer:
```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
```

**Step 3: run on X-Small** (uses Snowflake's built-in sample data, which trial accounts include; use `TPCH_SF10` instead of `TPCH_SF100` for a shorter run):
```sql
USE WAREHOUSE wh_xs;

SELECT l_returnflag, l_linestatus,
       SUM(l_quantity) AS qty, SUM(l_extendedprice) AS price,
       AVG(l_discount) AS avg_disc, COUNT(*) AS cnt
FROM snowflake_sample_data.tpch_sf100.lineitem
WHERE l_shipdate <= DATEADD(day, -90, '1998-12-01')
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
```

**Step 4: clear the warehouse cache, then run on Medium**
```sql
ALTER WAREHOUSE wh_xs SUSPEND;
USE WAREHOUSE wh_m;
-- run the exact same SELECT again
```

**Step 5: compare the time.** In Snowsight, go to Monitoring, then Query History, and note the **total duration** of each run. Or in SQL:
```sql
SELECT warehouse_name, warehouse_size, total_elapsed_time/1000 AS seconds
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(RESULT_LIMIT => 20))
WHERE query_text ILIKE '%lineitem%'
ORDER BY start_time DESC;
```

**Step 6: work out the credits.** Formula: `(seconds / 3600) x credits per hour`
- X-Small: seconds / 3600 x 1
- Medium: seconds / 3600 x 4

For the real billed numbers (updates after a delay of a few hours):
```sql
SELECT warehouse_name, SUM(credits_used_compute) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE warehouse_name IN ('WH_XS', 'WH_M')
GROUP BY warehouse_name;
```
The billed number can be higher than your formula because of the 60-second minimum and the startup time.

**Step 7: fill your table**
| | X-Small | Medium |
|---|---|---|
| Time (seconds) | ? | ? |
| Credits/hour | 1 | 4 |
| Credits used | ? | ? |
| Faster by | | ? times |

**Think about:** was Medium 4x faster? If it was only 2x faster, the query didn't scale perfectly, and Medium cost more for the same work.

**Clean up:** `DROP WAREHOUSE wh_xs; DROP WAREHOUSE wh_m;`
