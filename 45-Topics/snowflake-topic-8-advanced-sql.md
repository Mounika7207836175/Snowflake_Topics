# Snowflake Notes: Topic 8, Advanced SQL

## Part A: Clearing the doubts

### 1. ZEROIFNULL vs NVL
- `NVL(x, y)`: if x is NULL, give y. You choose **any** replacement.
- `ZEROIFNULL(x)`: if x is NULL, give **0**. A shortcut for `NVL(x, 0)`.
- `NULLIFZERO(x)`: the opposite. If x is 0, give NULL.
- `COALESCE(a, b, c)`: the first non-NULL value in the list.

```sql
SELECT NVL(NULL, 'unknown');     -- unknown
SELECT ZEROIFNULL(NULL);         -- 0
SELECT 100 / NULLIFZERO(0);      -- NULL instead of a divide-by-zero error
```
`a / NULLIFZERO(b)` avoids "division by zero" errors.

### 2. LISTAGG
Joins values from many rows into **one text value**. Roughly the reverse of FLATTEN.

| customer | order |
|---|---|
| Asha | 101 |
| Asha | 102 |
| Ravi | 103 |

```sql
SELECT customer,
       LISTAGG(order_id, ', ') WITHIN GROUP (ORDER BY order_id) AS orders
FROM orders
GROUP BY customer;
```
Result: Asha gets `101, 102`, Ravi gets `103`.
- `WITHIN GROUP (ORDER BY ...)` controls the order inside the text.
- `LISTAGG(DISTINCT col, ', ')` removes duplicates.
- Very long results hit a size limit, so avoid it on millions of values per group.

### 3. MEDIAN
The **middle value** when sorted. Half the values are below it, half above. NULLs are ignored.

*Why not AVG?* Salaries: 30k, 32k, 35k, 40k, 500k.
- AVG = 127.4k (pulled up by one huge salary, misleading)
- MEDIAN = 35k (the real "typical" salary)

Use MEDIAN when a few extreme values distort the average. It is heavier to calculate than AVG on huge tables, because it must sort the values.

### 4. APPROX_COUNT_DISTINCT
`COUNT(DISTINCT col)` is **exact**, but on billions of rows it is slow and memory-heavy. `APPROX_COUNT_DISTINCT(col)` gives a **very close estimate** (typically within about 1 to 2% error) much faster and cheaper.

*Example:* "unique visitors this month" on a dashboard with 5 billion click rows. 12.3 million vs 12.4 million doesn't change any decision, so use the approximate version.

Don't use it for billing, finance, or audits, where the exact number matters.

---

## Part B: Window functions

A window function calculates across a group of rows **but keeps every row**. (GROUP BY collapses rows. Window functions don't.)

```
function() OVER (PARTITION BY group_column ORDER BY sort_column)
```
- `PARTITION BY`: restart the calculation for each group (like GROUP BY, without collapsing).
- `ORDER BY`: the order inside the group.

Sample data used in the examples:
```sql
WITH sales AS (
  SELECT * FROM VALUES
    ('Asha','East','2024-01-05',100),
    ('Asha','East','2024-01-20',150),
    ('Asha','East','2024-02-03',150),
    ('Ravi','West','2024-01-10',200),
    ('Ravi','West','2024-02-14',120)
  AS t(emp, region, sale_date, amount)
)
```

### ROW_NUMBER, RANK, DENSE_RANK
| Function | Ties | Example for amounts 150, 150, 100 |
|---|---|---|
| `ROW_NUMBER()` | Always unique numbers | 1, 2, 3 |
| `RANK()` | Same rank for ties, **skips** next | 1, 1, 3 |
| `DENSE_RANK()` | Same rank for ties, **no skip** | 1, 1, 2 |

```sql
SELECT emp, amount,
  ROW_NUMBER() OVER (PARTITION BY emp ORDER BY amount DESC) AS rn,
  RANK()       OVER (PARTITION BY emp ORDER BY amount DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY emp ORDER BY amount DESC) AS drnk
FROM sales;
```
*Scenarios:* "top 2 sales per employee", "remove duplicate rows" (keep `ROW_NUMBER() = 1`).

### LAG and LEAD
Look at the **previous** (`LAG`) or **next** (`LEAD`) row.
```sql
SELECT emp, sale_date, amount,
  LAG(amount)  OVER (PARTITION BY emp ORDER BY sale_date) AS prev_amount,
  LEAD(amount) OVER (PARTITION BY emp ORDER BY sale_date) AS next_amount,
  amount - LAG(amount) OVER (PARTITION BY emp ORDER BY sale_date) AS change
FROM sales;
```
The first row has no previous, so `LAG` gives NULL (or a default: `LAG(amount, 1, 0)`). *Scenario:* month-over-month change.

### QUALIFY
Filters on window results directly, with no subquery:
```sql
SELECT emp, amount FROM sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY emp ORDER BY amount DESC) = 1;
```
Gives each employee's biggest sale.

---

## Part C: Running totals, step by step

**Think of a bank passbook.** Each line shows the transaction and the balance **so far**. A running total is the same: for each row, the sum of all rows up to that row.

### Example 1: one person's sales

| date | amount | running total |
|---|---|---|
| Jan 05 | 100 | 100 |
| Jan 20 | 150 | 250 (100 + 150) |
| Feb 03 | 150 | 400 (100 + 150 + 150) |

```sql
SELECT sale_date, amount,
       SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM (
  SELECT * FROM VALUES
    ('2024-01-05'::DATE, 100),
    ('2024-01-20'::DATE, 150),
    ('2024-02-03'::DATE, 150)
  AS t(sale_date, amount)
);
```
- `SUM(amount)`: add up amounts...
- `OVER (...)`: ...but **don't collapse the rows**. Keep every row and show the sum next to it.
- `ORDER BY sale_date`: "so far" means "in date order".

### Example 2: add PARTITION BY (a separate total for each person)

| emp | date | amount |
|---|---|---|
| Asha | Jan 05 | 100 |
| Ravi | Jan 10 | 200 |
| Asha | Jan 20 | 150 |
| Asha | Feb 03 | 150 |
| Ravi | Feb 14 | 120 |

```sql
SUM(amount) OVER (PARTITION BY emp ORDER BY sale_date)
```
`PARTITION BY emp` means: **restart the total for each person**.

| emp | date | amount | running total |
|---|---|---|---|
| Asha | Jan 05 | 100 | 100 |
| Asha | Jan 20 | 150 | 250 |
| Asha | Feb 03 | 150 | 400 |
| Ravi | Jan 10 | 200 | 200 |
| Ravi | Feb 14 | 120 | 320 |

### Example 3: remove PARTITION BY (one total for everyone)
```sql
SUM(amount) OVER (ORDER BY sale_date)
```
All 5 rows in date order: 100, 100+200=**300**, 300+150=**450**, 450+150=**600**, 600+120=**720**.

### Example 4: the frame (which rows are included)
By default the total includes everything from the first row to the current row. You can say it explicitly:
```sql
SUM(amount) OVER (PARTITION BY emp ORDER BY sale_date
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```
- `UNBOUNDED PRECEDING` = from the very first row
- `CURRENT ROW` = up to this row

Change the frame and you get a **moving average**. "Average of this row and the one before it" for Asha:
```sql
AVG(amount) OVER (PARTITION BY emp ORDER BY sale_date
                  ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)
```

| amount | what it averages | result |
|---|---|---|
| 100 | just 100 (nothing before) | 100 |
| 150 | 100, 150 | 125 |
| 150 | 150, 150 | 150 |

### Example 5: real scenario, bank balance
Transactions: +1000, -200, +500, -300.
```sql
SELECT txn_date, amount,
       SUM(amount) OVER (ORDER BY txn_date) AS balance
FROM (
  SELECT * FROM VALUES
    ('2024-01-01'::DATE, 1000),
    ('2024-01-05'::DATE, -200),
    ('2024-01-09'::DATE,  500),
    ('2024-01-12'::DATE, -300)
  AS t(txn_date, amount)
);
```
Balance column: **1000, 800, 1300, 1000**.

---

## Part D: LATERAL FLATTEN, step by step

### Step 0: the data
A table where one column holds JSON. Each customer's JSON has a **list** of order numbers.

```sql
CREATE OR REPLACE TEMPORARY TABLE customer_events AS
SELECT 1 AS id, PARSE_JSON('{"name":"Asha","orders":[101,102,103]}') AS data
UNION ALL
SELECT 2, PARSE_JSON('{"name":"Ravi","orders":[104]}')
UNION ALL
SELECT 3, PARSE_JSON('{"name":"Meena","orders":[]}');
```

| id | data |
|---|---|
| 1 | name Asha, orders [101, 102, 103] |
| 2 | name Ravi, orders [104] |
| 3 | name Meena, orders [] (empty list) |

**Goal:** one row per order number.

### Step 1: the query
```sql
SELECT c.id,
       c.data:name::STRING AS name,
       f.index,
       f.value::INT        AS order_id
FROM customer_events c,
     LATERAL FLATTEN(input => c.data:orders) f;
```

### Step 2: each piece in plain words
| Piece | Meaning |
|---|---|
| `FROM customer_events c` | Our table, nicknamed `c` |
| `,` (comma) | "Combine each row with whatever comes next" |
| `LATERAL` | "For **each row** of `c`, run the next part using **that row's** data" |
| `FLATTEN(...)` | "Split a list into separate rows" |
| `input => c.data:orders` | "The list to split is the `orders` list inside this row's JSON" |
| `f` | Nickname for the flatten output |
| `f.value` | The item itself (101, 102...) |
| `f.index` | The item's position, starting at 0 |

### Step 3: what happens row by row
- **Row 1 (Asha):** list has 3 items, so 3 output rows. Asha's `id` and `name` are repeated on each.
- **Row 2 (Ravi):** list has 1 item, so 1 output row.
- **Row 3 (Meena):** list is empty, so **0 rows** (she disappears).

| id | name | index | order_id |
|---|---|---|---|
| 1 | Asha | 0 | 101 |
| 1 | Asha | 1 | 102 |
| 1 | Asha | 2 | 103 |
| 2 | Ravi | 0 | 104 |

### Step 4: keep Meena with `OUTER => TRUE`
```sql
LATERAL FLATTEN(input => c.data:orders, OUTER => TRUE) f
```
Now Meena appears with NULL in `index` and `order_id`. Use this when you must not lose customers who have empty lists.

### Example 2: a list of **objects** (the most common real case)
An order with line items:
```sql
WITH o AS (
  SELECT PARSE_JSON('{
    "order_id": 1,
    "items": [
      {"product":"Pen",  "qty":2, "price":10},
      {"product":"Book", "qty":1, "price":50}
    ]}') AS data
)
SELECT o.data:order_id::INT          AS order_id,
       f.value:product::STRING       AS product,
       f.value:qty::INT              AS qty,
       f.value:price::NUMBER         AS price,
       f.value:qty::INT * f.value:price::NUMBER AS line_total
FROM o,
     LATERAL FLATTEN(input => o.data:items) f;
```
Each `f.value` is a small object, so you reach inside it with `f.value:product`.

| order_id | product | qty | price | line_total |
|---|---|---|---|---|
| 1 | Pen | 2 | 10 | 20 |
| 1 | Book | 1 | 50 | 50 |

Now you can `SUM(line_total)` per product, like normal SQL.

### Example 3: flattening an **object** (not a list)
```sql
SELECT f.key, f.value
FROM (SELECT PARSE_JSON('{"city":"Pune","age":30}') AS data) t,
     LATERAL FLATTEN(input => t.data) f;
```
For an object, FLATTEN gives one row per key:

| key | value |
|---|---|
| city | "Pune" |
| age | 30 |

Remember: **a list gives `value` and `index`, an object gives `key` and `value`.**

---

## Part E: Set operators
Combine the results of two queries (same number of columns, compatible types).

| Operator | Result |
|---|---|
| `UNION` | All rows from both, **duplicates removed** |
| `UNION ALL` | All rows from both, duplicates **kept** (faster) |
| `INTERSECT` | Only rows that appear in **both** |
| `EXCEPT` (or `MINUS`) | Rows in the first that are **not** in the second |

```sql
SELECT customer_id FROM orders_2023
INTERSECT
SELECT customer_id FROM orders_2024;     -- customers who ordered in both years
```
Tip: prefer `UNION ALL` unless you truly need duplicates removed, because removing them costs extra work.

---

## Part F: Sampling
Get a random slice of a large table, for quick testing.
```sql
SELECT * FROM orders SAMPLE (1);                  -- about 1% of rows
SELECT * FROM orders SAMPLE (1000 ROWS);          -- exactly 1000 rows
SELECT * FROM orders SAMPLE (1) REPEATABLE (42);  -- same sample every run (with a seed)
```
- **Row-level** (default, also called BERNOULLI): each row picked independently, more random.
- **Block-level** (`SAMPLE SYSTEM (1)`): picks whole micro-partitions, faster but less random.

---

## Part G: Recursive CTE, traced round by round

A CTE that **refers to itself**, used for hierarchies (managers and reports) and sequences. It has two queries joined by `UNION ALL`: an **anchor** (the starting rows) and a **recursive part** that uses the CTE itself.

### The data

| id | name | manager_id |
|---|---|---|
| 1 | Priya | NULL (the boss) |
| 2 | Arun | 1 |
| 3 | Divya | 1 |
| 4 | Karthik | 2 |
| 5 | Meena | 2 |
| 6 | Ravi | 4 |

As a tree:
```
Priya
 ├── Arun
 │    ├── Karthik
 │    │     └── Ravi
 │    └── Meena
 └── Divya
```

### The query
```sql
WITH RECURSIVE org AS (
  SELECT id, name, name AS path, 1 AS level
  FROM employees
  WHERE manager_id IS NULL                    -- ANCHOR

  UNION ALL

  SELECT e.id, e.name, o.path || ' > ' || e.name, o.level + 1
  FROM employees e
  JOIN org o ON e.manager_id = o.id           -- RECURSIVE PART
)
SELECT * FROM org ORDER BY level;
```
(`||` joins text together.)

### The rounds
**Round 0, anchor (runs once):** find the boss. Result so far: *Priya, level 1.*

**Round 1:** the recursive part looks **only at the rows from the previous round** (Priya, id 1) and asks: "who has `manager_id = 1`?" Found: Arun and Divya, level 2.

**Round 2:** previous round = Arun (id 2), Divya (id 3). "Who has `manager_id` 2 or 3?" Found: Karthik and Meena (reports of Arun), level 3. Divya has no reports.

**Round 3:** previous round = Karthik (4), Meena (5). "Who reports to 4 or 5?" Found: Ravi (reports to Karthik), level 4.

**Round 4:** previous round = Ravi (6). Nobody reports to 6, so **no new rows, so it stops.**

**Final answer = all rounds stacked together:**

| name | level | path |
|---|---|---|
| Priya | 1 | Priya |
| Arun | 2 | Priya > Arun |
| Divya | 2 | Priya > Divya |
| Karthik | 3 | Priya > Arun > Karthik |
| Meena | 3 | Priya > Arun > Meena |
| Ravi | 4 | Priya > Arun > Karthik > Ravi |

**Key idea:** in the recursive part, `o` means "the rows produced in the **previous round**", and `e` means "all employees". The `ON` condition connects "people found last round" to "their direct reports". That is the loop.

### Example 2: generate a list of dates
```sql
WITH RECURSIVE d AS (
  SELECT '2024-01-01'::DATE AS dt                  -- anchor: start date
  UNION ALL
  SELECT DATEADD(day, 1, dt) FROM d                -- add one day to last round's date
  WHERE dt < '2024-01-07'                          -- stop condition
)
SELECT * FROM d;
```
Round 0: Jan 1. Round 1: Jan 2. ... Round 6: Jan 7. Round 7: Jan 7 is not less than Jan 7, so it stops. Result: **7 rows, Jan 1 to Jan 7.**

*Real use:* fill in missing dates, so a chart shows 0 sales for days with no sales.

Example 3, numbers 1 to 5:
```sql
WITH RECURSIVE n AS (
  SELECT 1 AS num
  UNION ALL
  SELECT num + 1 FROM n WHERE num < 5
)
SELECT * FROM n;     -- 1, 2, 3, 4, 5
```

**Always include a stop condition** (here `WHERE num < 5`, in the org chart "no more reports"), or the query never ends.

---

## Part H: MATCH_RECOGNIZE, built up slowly

**Analogy:** a regular expression finds patterns in text, like "a followed by b" in letters. MATCH_RECOGNIZE finds patterns in **rows, in order**.

### Example 1: stock price falls, then rises

Daily prices for stock A:

| day | price | compared to the day before |
|---|---|---|
| 1 | 10 | (nothing before) |
| 2 | 12 | UP |
| 3 | 11 | DOWN |
| 4 | 9 | DOWN |
| 5 | 8 | DOWN |
| 6 | 10 | UP |
| 7 | 13 | UP |

As labels: `UP, DOWN, DOWN, DOWN, UP, UP`. We want the pattern **"one or more DOWN followed by one or more UP"**. That matches **day 3 to day 7**.

```sql
SELECT *
FROM stock_prices
MATCH_RECOGNIZE (
  PARTITION BY symbol
  ORDER BY day
  MEASURES
    MATCH_NUMBER() AS match_no,
    FIRST(day)     AS start_day,
    LAST(day)      AS end_day
  ONE ROW PER MATCH
  PATTERN (down+ up+)
  DEFINE
    down AS price < LAG(price),
    up   AS price > LAG(price)
);
```

| Part | Meaning |
|---|---|
| `PARTITION BY symbol` | Look at each stock separately |
| `ORDER BY day` | Rows must be in day order (patterns need order) |
| `DEFINE` | Gives a **name** to each kind of row. `down` = today's price is lower than yesterday's. `up` = higher |
| `PATTERN (down+ up+)` | The shape to find. `+` means "one or more" |
| `MEASURES` | What columns to return about each match |
| `ONE ROW PER MATCH` | One summary row per pattern found (`ALL ROWS PER MATCH` gives every row inside it) |

Result:

| match_no | start_day | end_day |
|---|---|---|
| 1 | 3 | 7 |

Pattern symbols (same idea as regex): `+` one or more, `*` zero or more, `?` optional, `|` or, `( )` grouping.

### Example 2: three or more failed logins in a row

| user | time | status |
|---|---|---|
| A | 1 | OK |
| A | 2 | FAIL |
| A | 3 | FAIL |
| A | 4 | FAIL |
| A | 5 | FAIL |
| A | 6 | OK |
| A | 7 | FAIL |
| A | 8 | OK |

```sql
SELECT *
FROM logins
MATCH_RECOGNIZE (
  PARTITION BY user_name
  ORDER BY login_time
  MEASURES
    FIRST(login_time) AS first_fail,
    LAST(login_time)  AS last_fail
  ONE ROW PER MATCH
  PATTERN (fail fail fail+)
  DEFINE
    fail AS status = 'FAIL'
);
```
`fail fail fail+` means "a fail, another fail, and one or more fails" = **3 or more in a row**.

Result: user A, first_fail 2, last_fail 5. The single FAIL at time 7 is not matched, because it is only one in a row.

Caution: Snowflake's exact syntax for looking at the previous row inside `DEFINE` (`LAG` above) can differ between versions. If you get an error when you practice it, check the MATCH_RECOGNIZE page in Snowflake's docs for the current form.

*Real uses:* detect fraud patterns, find login-failure bursts, group clicks into sessions.

---

## Quick recap
1. `ZEROIFNULL` = `NVL(x, 0)`. `NULLIFZERO` avoids divide-by-zero. `LISTAGG` = rows to one text. `MEDIAN` = middle value, not distorted by extremes. `APPROX_COUNT_DISTINCT` = fast estimate.
2. Window functions keep all rows. `PARTITION BY` = groups, `ORDER BY` = order inside the group.
3. ROW_NUMBER unique, RANK skips after ties, DENSE_RANK doesn't skip. LAG = previous row, LEAD = next row.
4. Running total = `SUM(...) OVER (ORDER BY ...)`. PARTITION BY restarts it per group. The frame decides which rows to add.
5. LATERAL FLATTEN: for each row, split its list into rows. List gives `value`/`index`, object gives `key`/`value`. `OUTER => TRUE` keeps empty lists.
6. UNION removes duplicates, UNION ALL keeps them, INTERSECT = in both, EXCEPT = in first only.
7. SAMPLE for quick random slices.
8. Recursive CTE: anchor starts it, the recursive part repeats using only the previous round's rows, and it stops when nothing new appears.
9. MATCH_RECOGNIZE: `DEFINE` names row types, `PATTERN` is the shape to find, like regex for rows.

---

## Hands-on suggestions
On `snowflake_sample_data.tpch_sf1`:
1. Rank customers by total spend within each nation (`DENSE_RANK`).
2. Month-over-month revenue change using `LAG`.
3. Running total of revenue by month.
4. Customers who ordered in 1995 and 1996 (`INTERSECT`), and only in 1995 (`EXCEPT`).
5. A recursive CTE that generates the numbers 1 to 10.
6. Flatten the `customer_events` example above, then add `OUTER => TRUE` and compare the row counts.
