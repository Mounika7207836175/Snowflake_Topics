# Snowflake Notes: Topic 11, Semi-Structured Data

## Sample data: 3 orders as nested JSON

```
Order 1001
  customer: name Asha, address { city Pune, pin 411001 }
  items:    [ {sku PEN,  qty 2, price 10,  tags [office, blue]},
              {sku BOOK, qty 1, price 50,  tags [study]} ]
  payment:  { method UPI, paid true }

Order 1002
  customer: name Ravi, address { city Chennai, pin 600001 }
  items:    [ {sku PEN, qty 5, price 10,  tags []},
              {sku BAG, qty 1, price 400, tags [travel, black]} ]
  payment:  { method CARD, paid true }

Order 1003
  customer: name Meena, address { city Hyderabad }      <- no pin
  items:    []                                          <- empty list
  payment:  { method COD, paid false }
  coupon:   null                                        <- JSON null
```
An **object** `{ }` holds named values, an **array** `[ ]` holds a list, and they can nest inside each other many levels deep.

## 1. VARIANT, OBJECT, ARRAY
- **VARIANT:** a column type that can hold any JSON value. One whole order goes into one VARIANT cell.
- **OBJECT:** key-value pairs, like `customer`.
- **ARRAY:** an ordered list, like `items`.

## 2. Path notation: reaching inside the JSON

Assume the VARIANT column is called `v`.

| You write | Meaning | Result for order 1001 |
|---|---|---|
| `v:order_id` | Top-level key (colon after the column) | 1001 |
| `v:customer.name` | Go one level deeper with a dot | "Asha" |
| `v:customer:name` | A colon works too | "Asha" |
| `v['customer']['name']` | Bracket form | "Asha" |
| `v:customer.address.city` | Three levels deep | "Pune" |
| `v:items[0].sku` | First item of the list (**index starts at 0**) | "PEN" |
| `v:items[1].price` | Second item's price | 50 |

Rules:
- **Keys are case-sensitive.** `v:Order_ID` returns NULL.
- **A missing key returns NULL**, not an error. `v:customer.address.pin` for Meena is NULL, and so is `v:items[0]` when the list is empty.

## 3. Casting: turning VARIANT values into normal types

A value pulled from VARIANT is still a VARIANT, so text shows with **quotes** (`"Asha"`). Cast it with `::`.

```sql
v:customer.name::STRING          -- Asha  (no quotes)
v:order_id::INT                  -- 1001
v:items[0].price::NUMBER(10,2)   -- 10.00
v:payment.paid::BOOLEAN          -- TRUE
TO_DATE(v:order_date::STRING)    -- text to a real date
```
- If a value can't be converted (the text `abc` as `::INT`), you get an error.
- `TRY_TO_NUMBER(...)`, `TRY_TO_DATE(...)`, and similar return NULL instead of an error. Apply them to the text form: `TRY_TO_NUMBER(v:x::STRING)`.

## 4. JSON null vs a missing key

- **Missing key:** the key isn't there at all (`coupon` for orders 1001 and 1002). Result: SQL NULL.
- **JSON null:** the key is there with the value `null` (`coupon` for order 1003).

```sql
SELECT v:order_id::INT AS id,
       TYPEOF(v:coupon)  AS coupon_type,
       v:coupon::STRING  AS coupon
FROM raw_orders ORDER BY id;
```
| id | coupon_type | coupon |
|---|---|---|
| 1001 | NULL | NULL |
| 1002 | NULL | NULL |
| 1003 | NULL_VALUE | NULL |

`TYPEOF()` tells you what is inside a VARIANT (`INTEGER`, `VARCHAR`, `BOOLEAN`, `OBJECT`, `ARRAY`, `NULL_VALUE`...). Casting a JSON null with `::STRING` gives a normal SQL NULL. `TYPEOF` is useful when exploring unknown data.

## 5. PARSE_JSON and TRY_PARSE_JSON

JSON often arrives as **plain text** (for example, an API response saved in a VARCHAR column). `PARSE_JSON` turns that text into a real VARIANT you can query.

```sql
SELECT PARSE_JSON('{"name":"Asha","city":"Pune"}'):city::STRING;    -- Pune
```
- `PARSE_JSON` on broken text gives an **error**.
- `TRY_PARSE_JSON('{bad json')` gives **NULL** instead, so one bad row doesn't kill the whole query.

*Scenario:* a table `api_logs(id, response_text)` stores raw JSON as text. Query it with `PARSE_JSON(response_text):status::STRING`, or `TRY_PARSE_JSON` if some rows may be corrupt.

## 6. Building JSON: OBJECT_CONSTRUCT, ARRAY_CONSTRUCT, ARRAY_AGG

```sql
SELECT OBJECT_CONSTRUCT('name', 'Asha', 'city', 'Pune');          -- {"city":"Pune","name":"Asha"}
SELECT ARRAY_CONSTRUCT(10, 20, 30);                               -- [10,20,30]
```
(The key order in the output may differ from what you typed.)

- `OBJECT_CONSTRUCT` **leaves out keys whose value is NULL**. Use `OBJECT_CONSTRUCT_KEEP_NULL` to keep them.
- `ARRAY_AGG(col)` collects values from many rows into one array (an aggregate, used with GROUP BY).

*Scenario:* one JSON summary per order to send to an API:
```sql
SELECT OBJECT_CONSTRUCT('order', v:order_id, 'customer', v:customer.name,
                        'item_count', ARRAY_SIZE(v:items)) AS summary
FROM raw_orders;
```

Skus of each order in one array:
```sql
SELECT v:order_id::INT AS id,
       ARRAY_AGG(i.value:sku::STRING) WITHIN GROUP (ORDER BY i.index) AS skus
FROM raw_orders, LATERAL FLATTEN(input => v:items) i
GROUP BY 1;
```
Result: 1001 gives `["PEN","BOOK"]`, 1002 gives `["PEN","BAG"]`.

## 7. FLATTEN and LATERAL

Recap: `FROM table, LATERAL FLATTEN(input => table.column:list) f` gives **one row per list item**. A list gives `f.value` and `f.index`. An object gives `f.key` and `f.value`. `OUTER => TRUE` keeps rows whose list is empty.

**Nested lists.** Each item has its own `tags` list inside `items`. To get one row per tag, use **two LATERAL FLATTENs, one after the other**:
```sql
FROM raw_orders,
     LATERAL FLATTEN(input => v:items) i,          -- first level: items
     LATERAL FLATTEN(input => i.value:tags) t      -- second level: tags inside each item
```
The second FLATTEN reads from `i.value` (the item produced by the first one). Like unpacking a box that contains smaller boxes.

**RECURSIVE => TRUE** flattens **every level at once** and shows the `path` to each value. It is the best tool for **exploring JSON you've never seen**.

## 8. Querying nested JSON (filters)

```sql
-- unpaid orders
SELECT v:order_id::INT FROM raw_orders WHERE v:payment.paid::BOOLEAN = FALSE;          -- 1003

-- orders from Pune
SELECT v:order_id::INT FROM raw_orders WHERE v:customer.address.city::STRING = 'Pune'; -- 1001

-- orders with 2 or more items
SELECT v:order_id::INT FROM raw_orders WHERE ARRAY_SIZE(v:items) >= 2;                 -- 1001, 1002

-- orders containing a BAG (search inside the list)
SELECT DISTINCT v:order_id::INT
FROM raw_orders, LATERAL FLATTEN(input => v:items) i
WHERE i.value:sku::STRING = 'BAG';                                                    -- 1002
```

**When to keep VARIANT and when to flatten:**
- Keep the raw JSON in a VARIANT table (**nothing is lost** if the sender adds new fields).
- Build a **view or table with normal typed columns** on top (faster for reporting, easy for Power BI). This "raw layer plus flattened layer" is the standard pattern.

---

## Hands-on: load nested JSON, flatten arrays into rows

**Step 0: setup**
```sql
USE WAREHOUSE learn_wh;
CREATE DATABASE IF NOT EXISTS practice_db;
CREATE SCHEMA IF NOT EXISTS practice_db.lab;
USE SCHEMA practice_db.lab;
CREATE STAGE IF NOT EXISTS lab_stage;
```

**Step 1: create the nested JSON file inside the stage**
```sql
COPY INTO @lab_stage/json/
FROM (
  SELECT PARSE_JSON(j) FROM VALUES
    ($${"order_id":1001,"customer":{"name":"Asha","address":{"city":"Pune","pin":411001}},"items":[{"sku":"PEN","qty":2,"price":10,"tags":["office","blue"]},{"sku":"BOOK","qty":1,"price":50,"tags":["study"]}],"payment":{"method":"UPI","paid":true}}$$),
    ($${"order_id":1002,"customer":{"name":"Ravi","address":{"city":"Chennai","pin":600001}},"items":[{"sku":"PEN","qty":5,"price":10,"tags":[]},{"sku":"BAG","qty":1,"price":400,"tags":["travel","black"]}],"payment":{"method":"CARD","paid":true}}$$),
    ($${"order_id":1003,"customer":{"name":"Meena","address":{"city":"Hyderabad"}},"items":[],"payment":{"method":"COD","paid":false},"coupon":null}$$)
  AS t(j)
)
FILE_FORMAT = (TYPE = JSON COMPRESSION = GZIP)
OVERWRITE = TRUE;

LIST @lab_stage/json/;
```
Expected: one `.json.gz` file. (`$$ ... $$` lets you paste JSON containing quotes without escaping.)

**Step 2: load it into a VARIANT table**
```sql
CREATE OR REPLACE TABLE raw_orders (v VARIANT);

COPY INTO raw_orders
FROM @lab_stage/json/
FILE_FORMAT = (TYPE = JSON);

SELECT * FROM raw_orders;
```
Expected: **3 rows**, each cell holding one whole order. (If a file instead contained one big array `[ {...}, {...} ]`, add `STRIP_OUTER_ARRAY = TRUE` to the file format, to get one row per order.)

**Step 3: path notation**
```sql
SELECT v:order_id::INT                 AS order_id,
       v:customer.name::STRING         AS customer,
       v:customer.address.city::STRING AS city,
       v:customer.address.pin::INT     AS pin,
       v:items[0].sku::STRING          AS first_item,
       v:payment.method::STRING        AS pay_method
FROM raw_orders
ORDER BY order_id;
```
Expected:

| order_id | customer | city | pin | first_item | pay_method |
|---|---|---|---|---|---|
| 1001 | Asha | Pune | 411001 | PEN | UPI |
| 1002 | Ravi | Chennai | 600001 | PEN | CARD |
| 1003 | Meena | Hyderabad | NULL | NULL | COD |

**Step 4: flatten the items array into rows**
```sql
SELECT v:order_id::INT     AS order_id,
       i.index             AS item_pos,
       i.value:sku::STRING AS sku,
       i.value:qty::INT    AS qty,
       i.value:price::INT  AS price,
       i.value:qty::INT * i.value:price::INT AS line_total
FROM raw_orders,
     LATERAL FLATTEN(input => v:items) i
ORDER BY order_id, item_pos;
```
Expected: **4 rows**

| order_id | item_pos | sku | qty | price | line_total |
|---|---|---|---|---|---|
| 1001 | 0 | PEN | 2 | 10 | 20 |
| 1001 | 1 | BOOK | 1 | 50 | 50 |
| 1002 | 0 | PEN | 5 | 10 | 50 |
| 1002 | 1 | BAG | 1 | 400 | 400 |

Order 1003 is missing, because its list is empty.

**Step 5: keep order 1003 with OUTER**
Change the FLATTEN line to:
```sql
LATERAL FLATTEN(input => v:items, OUTER => TRUE) i
```
Expected: **5 rows**. The new row is order 1003 with NULL in the item columns.

**Step 6: revenue per order (including the empty order)**
```sql
SELECT v:order_id::INT AS order_id,
       ZEROIFNULL(SUM(i.value:qty::INT * i.value:price::INT)) AS revenue
FROM raw_orders,
     LATERAL FLATTEN(input => v:items, OUTER => TRUE) i
GROUP BY 1
ORDER BY 1;
```
Expected: 1001 gives **70**, 1002 gives **450**, 1003 gives **0**.

**Step 7: nested flatten (tags inside items)**
```sql
SELECT v:order_id::INT AS order_id,
       i.value:sku::STRING AS sku,
       t.value::STRING     AS tag
FROM raw_orders,
     LATERAL FLATTEN(input => v:items) i,
     LATERAL FLATTEN(input => i.value:tags) t
ORDER BY order_id, i.index, t.index;
```
Expected: **5 rows**

| order_id | sku | tag |
|---|---|---|
| 1001 | PEN | office |
| 1001 | PEN | blue |
| 1001 | BOOK | study |
| 1002 | BAG | travel |
| 1002 | BAG | black |

Order 1002's PEN has an empty `tags` list, so it disappears. Try adding `OUTER => TRUE` to the second FLATTEN, and the PEN row appears with a NULL tag.

**Step 8: explore the structure with RECURSIVE**
```sql
SELECT f.path, TYPEOF(f.value) AS type, f.value
FROM raw_orders,
     LATERAL FLATTEN(input => v, RECURSIVE => TRUE) f
WHERE v:order_id = 1001
ORDER BY f.path;
```
Expected: about **24 rows**, listing paths such as `customer.address.city`, `items[0].sku`, `items[0].tags[1]`, and `payment.paid`, with each value's type.

**Step 9: expose it as a normal table for reporting**
```sql
CREATE OR REPLACE VIEW orders_flat AS
SELECT v:order_id::INT                 AS order_id,
       v:customer.name::STRING         AS customer,
       v:customer.address.city::STRING AS city,
       i.value:sku::STRING             AS sku,
       i.value:qty::INT                AS qty,
       i.value:price::INT              AS price
FROM raw_orders,
     LATERAL FLATTEN(input => v:items) i;

SELECT sku, SUM(qty) AS units FROM orders_flat GROUP BY sku ORDER BY units DESC;
```
Expected: PEN **7**, BOOK **1**, BAG **1**.

**Check yourself**
1. Why does `v:items[0]` give NULL for order 1003? (The list is empty, and a missing element returns NULL.)
2. What is the difference between a JSON `null` and a missing key? (JSON null is a key with the value null, a missing key is not there at all. After casting, both become SQL NULL.)
3. Why do two FLATTENs appear in Step 7? (One per list level: items first, then the tags inside each item.)
4. In Step 4, why is the column `i.value:sku`, not `i:sku`? (`i.value` is the item itself, and you reach inside it with a colon.)

---

## Quick recap
1. VARIANT holds any JSON. Paths use `:`, `.`, and `[index]` (starting at 0). Keys are case-sensitive, and missing keys return NULL.
2. Cast with `::TYPE`. Without a cast, text shows with quotes.
3. `PARSE_JSON` turns JSON text into VARIANT. `TRY_PARSE_JSON` returns NULL for broken text.
4. `OBJECT_CONSTRUCT` and `ARRAY_CONSTRUCT` build JSON. `ARRAY_AGG` collects rows into a list.
5. `LATERAL FLATTEN` turns a list into rows. Chain two for nested lists. `OUTER => TRUE` keeps empty lists. `RECURSIVE => TRUE` explores every level.
6. Standard pattern: raw VARIANT table first, flattened typed view or table on top.
