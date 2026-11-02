# Snowflake Practice Labs: Topic 8 (Advanced SQL) and Topic 9 (Loading Data)

**How to use this file**
- Run everything in a Snowsight worksheet, in order. Later labs reuse tables from earlier ones.
- For each lab: read the scenario, **try writing the query yourself first**, then compare with the answer below it.
- Every lab has an "Expected" line. If your result is different, paste your query and result into the chat and we'll find the problem together.
- Nothing here depends on objects from older practice. Everything is created fresh.

---

## Lab 0: Setup (run once)

```sql
CREATE WAREHOUSE IF NOT EXISTS learn_wh
  WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE INITIALLY_SUSPENDED = TRUE;
USE WAREHOUSE learn_wh;

CREATE DATABASE IF NOT EXISTS practice_db;
CREATE SCHEMA IF NOT EXISTS practice_db.lab;
USE SCHEMA practice_db.lab;

SELECT CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```
Expected: `LEARN_WH`, `PRACTICE_DB`, `LAB`.

---

# PART A: TOPIC 8, ADVANCED SQL

## Data for Part A

```sql
CREATE OR REPLACE TABLE sales (
  sale_id INT, emp STRING, region STRING, sale_date DATE, amount INT
);

INSERT INTO sales VALUES
 (1,  'Asha',  'East', '2024-01-05', 100),
 (2,  'Asha',  'East', '2024-01-20', 150),
 (3,  'Asha',  'East', '2024-02-03', 150),
 (4,  'Ravi',  'West', '2024-01-10', 200),
 (5,  'Ravi',  'West', '2024-02-14', 120),
 (6,  'Meena', 'East', '2024-01-15', 300),
 (7,  'Meena', 'East', '2024-02-10', 250),
 (8,  'Meena', 'East', '2024-03-01',  50),
 (9,  'Kiran', 'West', '2024-01-25', 180),
 (10, 'Kiran', 'West', '2024-02-18', 220),
 (11, 'Kiran', 'West', '2024-03-12',  90),
 (12, 'Ravi',  'West', '2024-03-05', 160);

CREATE OR REPLACE TABLE team (emp STRING);
INSERT INTO team VALUES ('Asha'),('Ravi'),('Meena'),('Kiran'),('Divya');

SELECT COUNT(*) AS row_count, SUM(amount) AS total FROM sales;
```
Expected: **12** rows, total **1970**. (`team` has Divya, who has no sales. We'll use her later.)

---

## Lab 8.1: Sales leaderboard (ROW_NUMBER, RANK, DENSE_RANK)

**Scenario:** the sales manager wants each rep's sales ranked from highest to lowest, restarting for every rep. Ties must share a rank with no gaps.

**Task:** show `emp`, `amount`, and all three ranking functions side by side, so you can see the difference.

```sql
SELECT emp, amount,
  ROW_NUMBER() OVER (PARTITION BY emp ORDER BY amount DESC) AS rn,
  RANK()       OVER (PARTITION BY emp ORDER BY amount DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY emp ORDER BY amount DESC) AS drnk
FROM sales
ORDER BY emp, amount DESC;
```
**Expected (Asha's rows):**

| emp | amount | rn | rnk | drnk |
|---|---|---|---|---|
| Asha | 150 | 1 | 1 | 1 |
| Asha | 150 | 2 | 1 | 1 |
| Asha | 100 | 3 | 3 | 2 |

**Check yourself:** which column answers the manager's request? (`drnk`, because ties share a rank and there's no gap.) Why does `rnk` jump from 1 to 3? (RANK skips after ties.)

---

## Lab 8.2: Best sale per rep, and top 2 per region (QUALIFY)

**Scenario:** HR gives a bonus to each rep for their single best sale. Management also wants the top 2 sales in each region.

**Task A:** one row per rep: their best sale. If two sales tie, take the earlier date.
```sql
SELECT emp, sale_date, amount
FROM sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY emp ORDER BY amount DESC, sale_date) = 1
ORDER BY emp;
```
**Expected:** Asha 2024-01-20 150, Kiran 2024-02-18 220, Meena 2024-01-15 300, Ravi 2024-01-10 200.

**Task B:** top 2 sales per region.
```sql
SELECT region, emp, amount
FROM sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) <= 2
ORDER BY region, amount DESC;
```
**Expected:** East: Meena 300, Meena 250. West: Kiran 220, Ravi 200.

**Check yourself:** rewrite Task A **without** QUALIFY (use a subquery or CTE). It must give the same result.

---

## Lab 8.3: Change from the previous sale (LAG and LEAD)

**Scenario:** the manager asks "for each sale, did the rep do better or worse than their previous sale?" and "how many days until the rep's next sale?"

```sql
SELECT emp, sale_date, amount,
  LAG(amount) OVER (PARTITION BY emp ORDER BY sale_date)                    AS prev_amount,
  amount - LAG(amount) OVER (PARTITION BY emp ORDER BY sale_date)           AS change,
  DATEDIFF(day, sale_date,
           LEAD(sale_date) OVER (PARTITION BY emp ORDER BY sale_date))      AS days_to_next_sale
FROM sales
ORDER BY emp, sale_date;
```
**Expected (Meena):**

| sale_date | amount | prev_amount | change | days_to_next_sale |
|---|---|---|---|---|
| 2024-01-15 | 300 | NULL | NULL | 26 |
| 2024-02-10 | 250 | 300 | -50 | 20 |
| 2024-03-01 | 50 | 250 | -200 | NULL |

**Check yourself:** why is the first row's `prev_amount` NULL? Change it to show 0 instead (hint: `LAG(amount, 1, 0)`).

---

## Lab 8.4: Running totals (rep, company, moving average)

**Scenario A:** finance wants each rep's cumulative sales over time.
```sql
SELECT emp, sale_date, amount,
  SUM(amount) OVER (PARTITION BY emp ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM sales
ORDER BY emp, sale_date;
```
**Expected (running_total):** Asha 100, 250, 400. Kiran 180, 400, 490. Meena 300, 550, 600. Ravi 200, 320, 480.

**Scenario B:** the CEO wants the **company-wide** cumulative total by date. (Remove `PARTITION BY`.)
```sql
SELECT sale_date, emp, amount,
  SUM(amount) OVER (ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS company_running_total
FROM sales
ORDER BY sale_date;
```
**Expected (company_running_total, in date order):** 100, 300, 600, 750, 930, 1080, 1330, 1450, 1670, 1720, 1880, **1970**. The last value must equal the total from Lab 0.

**Scenario C:** smooth out the ups and downs with a moving average of each rep's current and previous sale.
```sql
SELECT emp, sale_date, amount,
  ROUND(AVG(amount) OVER (PARTITION BY emp ORDER BY sale_date
                          ROWS BETWEEN 1 PRECEDING AND CURRENT ROW), 1) AS moving_avg
FROM sales
ORDER BY emp, sale_date;
```
**Expected (Meena):** 300.0, 275.0, 150.0.

**Check yourself:** what changes if you remove `PARTITION BY emp` from Scenario A? (One total across all reps instead of one per rep.)

---

## Lab 8.5: Month-over-month growth

**Scenario:** the monthly business review needs revenue per month and the % change from the previous month.

```sql
WITH monthly AS (
  SELECT DATE_TRUNC('month', sale_date) AS month, SUM(amount) AS revenue
  FROM sales
  GROUP BY 1
)
SELECT month, revenue,
  LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
  ROUND(100 * (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIFZERO(LAG(revenue) OVER (ORDER BY month)), 2) AS pct_change
FROM monthly
ORDER BY month;
```
**Expected:**

| month | revenue | prev_revenue | pct_change |
|---|---|---|---|
| 2024-01-01 | 930 | NULL | NULL |
| 2024-02-01 | 740 | 930 | -20.43 |
| 2024-03-01 | 300 | 740 | -59.46 |

**Check yourself:** why `NULLIFZERO` in the denominator? (It stops a divide-by-zero error if a previous month had 0 revenue.)

---

## Lab 8.6: The five doubts (ZEROIFNULL, NVL, LISTAGG, MEDIAN, APPROX_COUNT_DISTINCT)

**Scenario A:** a report lists every team member with their total sales. Divya has no sales, and the report must show 0, not blank.
```sql
SELECT t.emp,
       SUM(s.amount)              AS total_raw,
       ZEROIFNULL(SUM(s.amount))  AS total_clean,
       NVL(SUM(s.amount), -1)     AS total_nvl
FROM team t
LEFT JOIN sales s ON t.emp = s.emp
GROUP BY t.emp
ORDER BY t.emp;
```
**Expected (Divya):** `total_raw` NULL, `total_clean` 0, `total_nvl` -1.

**Scenario B:** a one-line summary per rep: all amounts in date order, the median, and the average.
```sql
SELECT emp,
  LISTAGG(amount::STRING, ', ') WITHIN GROUP (ORDER BY sale_date) AS amounts,
  MEDIAN(amount)                                                  AS median_amount,
  ROUND(AVG(amount), 2)                                           AS avg_amount
FROM sales
GROUP BY emp
ORDER BY emp;
```
**Expected:**

| emp | amounts | median_amount | avg_amount |
|---|---|---|---|
| Asha | 100, 150, 150 | 150 | 133.33 |
| Kiran | 180, 220, 90 | 180 | 163.33 |
| Meena | 300, 250, 50 | 250 | 200.00 |
| Ravi | 200, 120, 160 | 160 | 160.00 |

**Scenario C:** exact vs approximate unique count.
```sql
SELECT COUNT(DISTINCT emp) AS exact_reps, APPROX_COUNT_DISTINCT(emp) AS approx_reps FROM sales;
```
**Expected:** both **4** (on a tiny table they match. The difference only appears on billions of rows).

**Check yourself:** add one more row to `sales` with a huge amount for Asha (for example 5000) and compare her `MEDIAN` and `AVG` again. Which one barely moves?

---

## Lab 8.7: FLATTEN, from simple to real

### Data
```sql
CREATE OR REPLACE TABLE customer_events AS
SELECT 1 AS id, PARSE_JSON('{"name":"Asha","orders":[101,102,103]}') AS data
UNION ALL SELECT 2, PARSE_JSON('{"name":"Ravi","orders":[104]}')
UNION ALL SELECT 3, PARSE_JSON('{"name":"Meena","orders":[]}');

CREATE OR REPLACE TABLE orders_json AS
SELECT 1 AS order_id, PARSE_JSON('{"customer":"Asha","items":[{"product":"Pen","qty":2,"price":10},{"product":"Book","qty":1,"price":50}]}') AS data
UNION ALL
SELECT 2, PARSE_JSON('{"customer":"Ravi","items":[{"product":"Pen","qty":5,"price":10},{"product":"Bag","qty":1,"price":400}]}');
```

### 8.7a: one row per order number
```sql
SELECT c.id, c.data:name::STRING AS name, f.index, f.value::INT AS order_id
FROM customer_events c,
     LATERAL FLATTEN(input => c.data:orders) f;
```
**Expected:** 4 rows (Asha 101, 102, 103 and Ravi 104). Meena is missing.

### 8.7b: keep customers with empty lists
```sql
SELECT c.id, c.data:name::STRING AS name, f.value::INT AS order_id
FROM customer_events c,
     LATERAL FLATTEN(input => c.data:orders, OUTER => TRUE) f;
```
**Expected:** 5 rows. Meena appears with `order_id` NULL.

### 8.7c: number of orders per customer, without FLATTEN
```sql
SELECT data:name::STRING AS name, ARRAY_SIZE(data:orders) AS order_count
FROM customer_events;
```
**Expected:** Asha 3, Ravi 1, Meena 0.

### 8.7d: put the orders back into one text value (FLATTEN then LISTAGG)
```sql
SELECT c.data:name::STRING AS name,
       LISTAGG(f.value::STRING, ', ') WITHIN GROUP (ORDER BY f.index) AS order_list
FROM customer_events c,
     LATERAL FLATTEN(input => c.data:orders) f
GROUP BY 1;
```
**Expected:** Asha `101, 102, 103`, Ravi `104`.

### 8.7e (real scenario): revenue per product from order line items
**Scenario:** each order is a JSON document with a list of items. The business wants revenue per product.
```sql
SELECT f.value:product::STRING AS product,
       SUM(f.value:qty::INT * f.value:price::INT) AS revenue
FROM orders_json o,
     LATERAL FLATTEN(input => o.data:items) f
GROUP BY 1
ORDER BY revenue DESC;
```
**Expected:** Bag 400, Pen 70 (20 + 50), Book 50.

### 8.7f: flatten an object (keys instead of positions)
```sql
SELECT f.key, f.value
FROM (SELECT PARSE_JSON('{"city":"Pune","age":30}') AS data) t,
     LATERAL FLATTEN(input => t.data) f;
```
**Expected:** two rows, `city` / "Pune" and `age` / 30.

**Check yourself:** in 8.7a the column is `f.index`, and in 8.7f it is `f.key`. When do you get which? (List gives `index`, object gives `key`.)

---

## Lab 8.8: Set operators

**Scenario:** the manager asks which reps were active in January, which were active in March, and who disappeared.

```sql
-- Active in both January and March
SELECT emp FROM sales WHERE MONTH(sale_date) = 1
INTERSECT
SELECT emp FROM sales WHERE MONTH(sale_date) = 3;
```
**Expected:** Kiran, Meena, Ravi.

```sql
-- Active in January but not in March
SELECT emp FROM sales WHERE MONTH(sale_date) = 1
EXCEPT
SELECT emp FROM sales WHERE MONTH(sale_date) = 3;
```
**Expected:** Asha.

```sql
-- UNION removes duplicates, UNION ALL keeps them
SELECT emp FROM sales WHERE MONTH(sale_date) = 1
UNION
SELECT emp FROM sales WHERE MONTH(sale_date) = 3;      -- 4 rows

SELECT emp FROM sales WHERE MONTH(sale_date) = 1
UNION ALL
SELECT emp FROM sales WHERE MONTH(sale_date) = 3;      -- 8 rows
```
**Expected:** 4 rows with `UNION`, 8 rows with `UNION ALL` (5 January rows + 3 March rows).

**Check yourself:** why 8 and not 4? (UNION ALL keeps duplicates, such as Asha appearing twice in January.)

---

## Lab 8.9: Sampling

**Scenario:** you want to test a query quickly on a small random piece of a big table, using Snowflake's built-in sample data.

```sql
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders;                  -- all rows: 1,500,000
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders SAMPLE (1);       -- about 15,000
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders SAMPLE (1);       -- a different number
SELECT COUNT(*) FROM (SELECT * FROM snowflake_sample_data.tpch_sf1.orders SAMPLE (1000 ROWS));  -- exactly 1000
SELECT COUNT(*) FROM snowflake_sample_data.tpch_sf1.orders SAMPLE (1) REPEATABLE (42);          -- run twice: same number
```
**Expected:** the two `SAMPLE (1)` counts are close to 15,000 but not identical. With `REPEATABLE (42)`, both runs give the same number.

**Check yourself:** when would you use the same-sample (seed) version? (When you need repeatable test results.)

---

## Lab 8.10: Recursive CTE (org chart and missing dates)

### 8.10a: org chart
```sql
CREATE OR REPLACE TABLE employees (id INT, name STRING, manager_id INT);
INSERT INTO employees VALUES
 (1,'Priya',NULL),(2,'Arun',1),(3,'Divya',1),(4,'Karthik',2),(5,'Meena',2),(6,'Ravi',4);

WITH RECURSIVE org AS (
  SELECT id, name, name AS path, 1 AS level
  FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, o.path || ' > ' || e.name, o.level + 1
  FROM employees e
  JOIN org o ON e.manager_id = o.id
)
SELECT name, level, path FROM org ORDER BY level, name;
```
**Expected:** 6 rows. Ravi is at level 4 with path `Priya > Arun > Karthik > Ravi`.

**Try it:** add `(7,'Sana',6)` to `employees` and run again. Sana should appear at level 5.

### 8.10b (real scenario): show every day of January, including days with no sales
**Scenario:** a chart must show 0 on days without sales instead of skipping them.
```sql
WITH RECURSIVE d AS (
  SELECT '2024-01-01'::DATE AS dt
  UNION ALL
  SELECT DATEADD(day, 1, dt) FROM d WHERE dt < '2024-01-31'
)
SELECT d.dt, COALESCE(SUM(s.amount), 0) AS revenue
FROM d
LEFT JOIN sales s ON s.sale_date = d.dt
GROUP BY d.dt
ORDER BY d.dt;
```
**Expected:** **31 rows**. Only 5 have revenue above 0: Jan 5 (100), Jan 10 (200), Jan 15 (300), Jan 20 (150), Jan 25 (180).

**Check yourself:** what stops the recursion in 8.10a? (No employee has the previous round's person as manager.) In 8.10b? (The `WHERE dt < '2024-01-31'` condition.)

---

## Lab 8.11 (optional challenge): MATCH_RECOGNIZE

### Data
```sql
CREATE OR REPLACE TABLE stock_prices (symbol STRING, trade_day INT, price INT);
INSERT INTO stock_prices VALUES
 ('A',1,10),('A',2,12),('A',3,11),('A',4,9),('A',5,8),('A',6,10),('A',7,13);

CREATE OR REPLACE TABLE logins (user_name STRING, login_time INT, status STRING);
INSERT INTO logins VALUES
 ('A',1,'OK'),('A',2,'FAIL'),('A',3,'FAIL'),('A',4,'FAIL'),('A',5,'FAIL'),
 ('A',6,'OK'),('A',7,'FAIL'),('A',8,'OK');
```

### 8.11a: price falls, then rises
The direction (UP or DOWN) is worked out first with `LAG` in a normal window function. Then MATCH_RECOGNIZE looks for the shape.
```sql
WITH p AS (
  SELECT symbol, trade_day, price,
    CASE WHEN price > LAG(price) OVER (PARTITION BY symbol ORDER BY trade_day) THEN 'UP'
         WHEN price < LAG(price) OVER (PARTITION BY symbol ORDER BY trade_day) THEN 'DOWN'
    END AS dir
  FROM stock_prices
)
SELECT *
FROM p
MATCH_RECOGNIZE (
  PARTITION BY symbol
  ORDER BY trade_day
  MEASURES
    MATCH_NUMBER()   AS match_no,
    FIRST(trade_day) AS start_day,
    LAST(trade_day)  AS end_day
  ONE ROW PER MATCH
  PATTERN (d+ u+)
  DEFINE
    d AS dir = 'DOWN',
    u AS dir = 'UP'
);
```
**Expected:** one row: match 1, start_day **3**, end_day **7**.

### 8.11b: three or more failed logins in a row
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
  PATTERN (f f f+)
  DEFINE
    f AS status = 'FAIL'
);
```
**Expected:** one row: first_fail **2**, last_fail **5**. The single FAIL at time 7 is not matched.

If either query gives an error, paste the exact error message in the chat and we'll fix it together.

---

# PART B: TOPIC 9, LOADING DATA

## Lab 9.0: Setup, and creating the practice files with no uploads

You don't need to upload anything for the first labs. Snowflake can **write** files into a stage, and then you load them back, which is exactly what a real pipeline does with files from other systems.

```sql
CREATE OR REPLACE FILE FORMAT csv_ff
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE OR REPLACE STAGE lab_stage;      -- a named internal stage

SHOW STAGES IN SCHEMA practice_db.lab;  -- should list LAB_STAGE
```

Create a CSV file inside the stage from a query:
```sql
COPY INTO @lab_stage/emp/
FROM (
  SELECT * FROM VALUES
    (1,'Asha','Pune',50000),
    (2,'Ravi','Chennai',62000),
    (3,'Meena','Hyderabad, India',58000)
  AS t(id, name, city, salary)
)
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' COMPRESSION = GZIP)
HEADER = TRUE
OVERWRITE = TRUE;

LIST @lab_stage/emp/;
```
**Expected:** one file under `lab_stage/emp/`, ending in `.csv.gz`.

**Optional, using real PUT:** to practice `PUT` as well, save `employees.csv` on your laptop and use SnowSQL:
```sql
PUT file://C:\snowflake_lab\employees.csv @lab_stage/put_test/;
```
(Or in Snowsight: Data, then your database and schema, then Stages, then `LAB_STAGE`, then **+ Files**.)

---

## Lab 9.1: Internal stage load, start to finish

**Scenario:** a daily employee file lands in the stage. Load it into a table and verify it.

```sql
CREATE OR REPLACE TABLE emp (id INT, name STRING, city STRING, salary NUMBER);

COPY INTO emp
FROM @lab_stage/emp/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');

SELECT * FROM emp;
```
**Expected:** the COPY result shows 1 file, status `LOADED`, 3 rows loaded, 0 errors. `emp` has 3 rows, and Meena's city is **`Hyderabad, India` in one column** (the quotes protected the comma).

**Check yourself:** what would happen to Meena's row if the file format had no `FIELD_OPTIONALLY_ENCLOSED_BY`? (The city would be split in two at the comma, and the columns would shift.)

---

## Lab 9.2: Running the load twice (load history, FORCE, duplicates)

**Scenario:** a scheduler accidentally runs the same load job twice.

**Step 1:** run the exact same COPY again.
```sql
COPY INTO emp
FROM @lab_stage/emp/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');

SELECT COUNT(*) FROM emp;
```
**Expected:** the message says **0 files processed**, and the count is still **3**. Snowflake remembered the file and skipped it.

**Step 2:** force the reload.
```sql
COPY INTO emp
FROM @lab_stage/emp/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff')
FORCE = TRUE;

SELECT COUNT(*) FROM emp;
```
**Expected:** count is now **6** (duplicates).

**Step 3:** look at the history.
```sql
SELECT file_name, status, row_count, last_load_time
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'EMP',
  START_TIME => DATEADD(hour, -1, CURRENT_TIMESTAMP())));
```
**Expected:** two load entries for the same file.

**Step 4:** clean up and reload properly.
```sql
TRUNCATE TABLE emp;

COPY INTO emp
FROM @lab_stage/emp/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');

SELECT COUNT(*) FROM emp;
```
**Expected:** count is **3** again. If it says 0 files processed, add `FORCE = TRUE` once.

**Check yourself:** why does Snowflake skip files it already loaded? (To protect you from loading the same data twice.)

---

## Lab 9.3: A bad row (dry run, ON_ERROR, VALIDATE)

**Scenario:** a new file arrives where one salary says `abc` instead of a number.

**Step 1:** create the bad file inside the stage.
```sql
COPY INTO @lab_stage/bad/
FROM (
  SELECT * FROM VALUES
    (4,'Kiran','Delhi','abc'),
    (5,'Divya','Mumbai','70000')
  AS t(id, name, city, salary)
)
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' COMPRESSION = GZIP)
HEADER = TRUE
OVERWRITE = TRUE;
```

**Step 2:** dry run (loads nothing, only reports problems).
```sql
COPY INTO emp
FROM @lab_stage/bad/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff')
VALIDATION_MODE = RETURN_ERRORS;
```
**Expected:** one error row mentioning that `abc` is not a recognized number, with the file name and line number.

**Step 3:** try the real load with the default behavior.
```sql
COPY INTO emp
FROM @lab_stage/bad/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');

SELECT COUNT(*) FROM emp;
```
**Expected:** the load **fails** with a "Numeric value 'abc' is not recognized" error, and the count stays **3**. (This is the default, `ABORT_STATEMENT`.)

**Step 4:** load the good rows, skip the bad one.
```sql
COPY INTO emp
FROM @lab_stage/bad/
FILE_FORMAT = (FORMAT_NAME = 'csv_ff')
ON_ERROR = CONTINUE;

SELECT * FROM emp ORDER BY id;
```
**Expected:** status `PARTIALLY_LOADED`, 1 row loaded, 1 error seen. `emp` now has **4** rows (Divya is in, Kiran is not).

**Step 5:** see exactly which row was rejected.
```sql
SELECT * FROM TABLE(VALIDATE(emp, JOB_ID => '_last'));
```
**Expected:** the Kiran row with the error text.

**Check yourself:** when would you choose `SKIP_FILE` instead of `CONTINUE`? (When one bad row means the whole file is suspect, so you want to reject the entire file and review it.)

---

## Lab 9.4: Several files, PATTERN, and PURGE

**Scenario:** monthly sales files arrive. Load only January's file, then delete it from the stage so it never gets reprocessed.

**Step 1:** create two monthly files from the `sales` table (from Part A).
```sql
COPY INTO @lab_stage/sales_jan_
FROM (SELECT emp, amount FROM sales WHERE MONTH(sale_date) = 1)
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' COMPRESSION = GZIP)
HEADER = TRUE;

COPY INTO @lab_stage/sales_feb_
FROM (SELECT emp, amount FROM sales WHERE MONTH(sale_date) = 2)
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' COMPRESSION = GZIP)
HEADER = TRUE;

LIST @lab_stage;
```
**Expected:** among the files you see one starting with `sales_jan_` and one starting with `sales_feb_`.

**Step 2:** load only January, and purge it afterwards.
```sql
CREATE OR REPLACE TABLE sales_load (emp STRING, amount INT);

COPY INTO sales_load
FROM @lab_stage
PATTERN = '.*sales_jan_.*'
FILE_FORMAT = (FORMAT_NAME = 'csv_ff')
PURGE = TRUE;

SELECT COUNT(*) FROM sales_load;
LIST @lab_stage;
```
**Expected:** count is **5** (January rows). `LIST` no longer shows the January file, but the February file is still there.

**Step 3:** load February.
```sql
COPY INTO sales_load
FROM @lab_stage
PATTERN = '.*sales_feb_.*'
FILE_FORMAT = (FORMAT_NAME = 'csv_ff');

SELECT COUNT(*) FROM sales_load;
```
**Expected:** count is **9** (5 January + 4 February).

**Check yourself:** what does `PURGE = TRUE` protect you from? (Old files piling up in the stage, and accidental reprocessing.)

---

## Lab 9.5: Parquet (VARIANT, MATCH_BY_COLUMN_NAME, INFER_SCHEMA, transform in COPY)

**Scenario:** another team sends you data as Parquet files. Load it four different ways and compare.

**Step 1:** create Parquet files in the stage from the `sales` table.
```sql
COPY INTO @lab_stage/parq/
FROM (SELECT * FROM sales)
FILE_FORMAT = (TYPE = PARQUET)
HEADER = TRUE;

LIST @lab_stage/parq/;
```
**Expected:** one or more `.parquet` files.

Note: this trick can create CSV, JSON, and Parquet files, but **not** Avro or ORC. For Avro and ORC, everything below works the same, you only change `TYPE = AVRO` or `TYPE = ORC`.

**Step 2:** see what Snowflake reads inside the file.
```sql
CREATE OR REPLACE FILE FORMAT parquet_ff TYPE = PARQUET;

SELECT *
FROM TABLE(INFER_SCHEMA(
  LOCATION => '@lab_stage/parq/',
  FILE_FORMAT => 'parquet_ff'));
```
**Expected:** a list with the column names (`SALE_ID`, `EMP`, `REGION`, `SALE_DATE`, `AMOUNT`), their types, and nullable flags.

**Way 1: everything into a VARIANT column**
```sql
CREATE OR REPLACE TABLE raw_sales (v VARIANT);

COPY INTO raw_sales
FROM @lab_stage/parq/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff');

SELECT v:EMP::STRING AS emp, v:AMOUNT::INT AS amount FROM raw_sales LIMIT 5;
SELECT COUNT(*) FROM raw_sales;
```
**Expected:** 12 rows. **Try it:** change `v:EMP` to `v:emp` and see that it returns NULL. Keys are **case-sensitive**, and the unloaded column names are uppercase.

**Way 2: match columns by name into a real table**
```sql
CREATE OR REPLACE TABLE sales_p LIKE sales;

COPY INTO sales_p
FROM @lab_stage/parq/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT COUNT(*) FROM sales_p;
```
**Expected:** 12 rows, with real typed columns (`sale_date` is a DATE).

**Way 3: let Snowflake create the table from the file**
```sql
CREATE OR REPLACE TABLE sales_auto USING TEMPLATE (
  SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
  FROM TABLE(INFER_SCHEMA(
    LOCATION => '@lab_stage/parq/',
    FILE_FORMAT => 'parquet_ff')));

DESCRIBE TABLE sales_auto;

COPY INTO sales_auto
FROM @lab_stage/parq/
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT COUNT(*) FROM sales_auto;
```
**Expected:** the table has the 5 columns Snowflake discovered, and 12 rows.

**Way 4: pick and convert columns while loading**
```sql
CREATE OR REPLACE TABLE emp_amount (emp STRING, amount INT);

COPY INTO emp_amount
FROM (
  SELECT $1:EMP::STRING, $1:AMOUNT::INT
  FROM @lab_stage/parq/
)
FILE_FORMAT = (FORMAT_NAME = 'parquet_ff');

SELECT emp, SUM(amount) FROM emp_amount GROUP BY emp ORDER BY emp;
```
**Expected:** Asha 400, Kiran 490, Meena 600, Ravi 480.

**Check yourself:** which way would you pick for (a) a quick look at an unknown file (Way 1), (b) a production table with a fixed design (Way 2), (c) a file with 80 columns you don't want to type (Way 3), (d) needing only 2 of 20 columns (Way 4)?

---

## Lab 9.6: External stage with Azure (needs a free Azure account)

If you don't have an Azure account yet, skip this lab for now.

**Scenario:** files are dropped in an Azure storage container by another system, and Snowflake must read them without you uploading anything to Snowflake.

### Part 1: Azure side
1. In the Azure portal, create a **storage account** (for example `mystorageacct`) and a **container** (for example `snowflake-data`).
2. Create a small file on your laptop, `azure_emp.csv`:
   ```
   id,name,city,salary
   7,Anil,Kochi,48000
   8,Sita,Jaipur,53000
   ```
3. Open **Azure Storage Explorer**, sign in, go to Storage Accounts, then your account, then Blob Containers, then `snowflake-data`. Click **Upload**, **Upload Files**, and choose `azure_emp.csv`. (You can put it in a folder, such as `employees/`.)
4. Create access: right-click the container and choose **Get Shared Access Signature**. Tick **Read** and **List**, choose an expiry time a few hours ahead, click **Create**, and copy the **query string** (it starts with `?sv=`).

### Part 2: Snowflake side
```sql
CREATE OR REPLACE STAGE azure_stage
  URL = 'azure://mystorageacct.blob.core.windows.net/snowflake-data/employees/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<paste the query string here>')
  FILE_FORMAT = csv_ff;

LIST @azure_stage;
```
**Expected:** `LIST` shows `azure_emp.csv`.

```sql
CREATE OR REPLACE TABLE emp_ext (id INT, name STRING, city STRING, salary NUMBER);

COPY INTO emp_ext FROM @azure_stage;

SELECT * FROM emp_ext;
```
**Expected:** 2 rows (Anil and Sita).

### Part 3: if something fails
| Symptom | Likely cause |
|---|---|
| `LIST` returns nothing | Wrong URL or folder path, or the file is in a different folder |
| Access denied or authentication error | Expired SAS token, or Read/List not ticked |
| 0 files processed on COPY | The file was already loaded, or the stage path is wrong |

### Part 4 (read only, for interviews): storage integration
In real projects, an admin creates a **storage integration** instead of a SAS token, and an Azure admin approves it. The steps and commands are in your Topic 9 notes. Write them out once from memory, then compare with the notes.

---

## Cleanup (when you finish everything)
```sql
DROP DATABASE IF EXISTS practice_db;
DROP WAREHOUSE IF EXISTS learn_wh;
```

---

## Self-check questions (answers below)
1. Which ranking function skips numbers after a tie?
2. What does `PARTITION BY` do in a window function?
3. What does `OUTER => TRUE` do in FLATTEN?
4. Why does a second `COPY INTO` of the same file load 0 files?
5. What are three `ON_ERROR` options, and what does each do?
6. What does `VALIDATION_MODE = RETURN_ERRORS` do?
7. Name three ways to load a Parquet file into a table.
8. Why use a storage integration instead of a SAS token?

**Answers**
1. `RANK`.
2. It restarts the calculation for each group, without collapsing the rows.
3. It keeps rows whose list is empty, showing NULL for the item.
4. Snowflake remembers loaded files for 64 days and skips them. `FORCE = TRUE` overrides this.
5. `ABORT_STATEMENT` (default, stop on any error), `CONTINUE` (skip bad rows, load the rest), `SKIP_FILE` (skip any file containing an error).
6. A dry run: it reports errors without loading any data.
7. Into one VARIANT column; with `MATCH_BY_COLUMN_NAME`; with a `SELECT` inside the COPY. (Also: create the table with `INFER_SCHEMA` and `USING TEMPLATE`.)
8. No secret is stored in SQL, access is controlled by the Azure admin, and nothing expires on its own.
