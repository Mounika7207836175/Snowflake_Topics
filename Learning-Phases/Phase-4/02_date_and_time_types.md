# Snowflake: Date and Time Types (DATE, TIME and Date Functions)

---

## 1. DATE and TIME

### DATE: only the calendar date
A `DATE` stores **year, month and day**, with no time part. It always **displays** as `YYYY-MM-DD`.

```sql
CREATE TABLE ORDERS (order_id NUMBER, order_date DATE);
INSERT INTO ORDERS VALUES (1, '2025-12-25');
SELECT CURRENT_DATE();      -- today's date
```

### Converting text to a date
Snowflake can guess common formats, but guessing is risky: `'01/02/2025'` may be read as MM/DD/YYYY (January 2). Always give the format as the second argument. The format describes **how to read the input text**, not how the date is displayed.

```sql
SELECT TO_DATE('25-12-2025', 'DD-MM-YYYY');     -- 2025-12-25
SELECT TO_DATE('Dec 25 2025', 'MON DD YYYY');   -- 2025-12-25
```

| Piece | Meaning | Example |
|---|---|---|
| `YYYY` | 4-digit year | 2025 |
| `MM` | month number | 12 |
| `MON` | short month name | Dec |
| `MMMM` | full month name | December |
| `DD` | day | 25 |

### TO_DATE vs TRY_TO_DATE
The difference is about **invalid or unreadable values** (not NULL input).
```sql
SELECT TRY_TO_DATE('31-02-2025', 'DD-MM-YYYY');   -- NULL (no such date)
SELECT TO_DATE('31-02-2025', 'DD-MM-YYYY');       -- ERROR
```
- `TO_DATE`: fails the whole query on a bad value
- `TRY_TO_DATE`: returns NULL for that value and continues with the others

### Other ways to build a date
```sql
SELECT '2025-12-25'::DATE;              -- cast
SELECT DATE_FROM_PARTS(2025, 12, 25);   -- from year, month, day
```

### Adding days
```sql
SELECT '2025-12-25'::DATE + 7;    -- 2026-01-01
SELECT '2025-12-25'::DATE - 1;    -- 2025-12-24
```

### TIME: only the time of day
```sql
SELECT '14:30:45'::TIME;                      -- 14:30:45
SELECT TO_TIME('02:30 PM', 'HH12:MI AM');     -- 14:30:00
SELECT CURRENT_TIME();
```
- Format is `HH24:MI:SS`, with optional fractions of a second (`TIME(3)` for milliseconds)
- No date and no time zone

**Where each fits:** `DATE` for birthdays, order dates, report dates. `TIME` for shift or opening times (rare in pipelines).

---

## 2. Date Functions

### TO_CHAR: choose the display format (returns a string)
```sql
SELECT TO_CHAR('2025-12-25'::DATE, 'DD-MON-YYYY');   -- 25-DEC-2025
SELECT TO_CHAR('2025-12-25'::DATE, 'DD/MM/YYYY');    -- 25/12/2025
```

### Pulling out parts
```sql
SELECT YEAR('2025-12-25'::DATE);                    -- 2025
SELECT MONTH('2025-12-25'::DATE);                   -- 12
SELECT DAY('2025-12-25'::DATE);                     -- 25
SELECT DAYNAME('2025-12-25'::DATE);                 -- Thu
SELECT EXTRACT(QUARTER FROM '2025-12-25'::DATE);    -- 4
```

### DATEADD: add or subtract any unit
```sql
SELECT DATEADD('day', 10, '2025-12-25'::DATE);     -- 2026-01-04
SELECT DATEADD('year', -1, '2025-12-25'::DATE);    -- 2024-12-25
```
**Month math:** Snowflake keeps the **same day number** when it exists in the new month, and uses the **last valid day** only when it doesn't.
```sql
SELECT DATEADD('month', 1, '2025-01-12'::DATE);   -- 2025-02-12
SELECT DATEADD('month', 1, '2025-01-28'::DATE);   -- 2025-02-28
SELECT DATEADD('month', 1, '2025-01-31'::DATE);   -- 2025-02-28 (no Feb 31)
SELECT DATEADD('month', 1, '2024-01-31'::DATE);   -- 2024-02-29 (leap year)
```

### DATEDIFF: difference between two dates
```sql
SELECT DATEDIFF('day', '2025-12-01', '2025-12-25');    -- 24
SELECT DATEDIFF('month', '2025-01-31', '2025-02-01');  -- 1
```
- Order is **(unit, start, end)**
- It counts **boundaries crossed**, not full units. Jan 31 to Feb 1 is one day apart, but the month changed, so the month difference is 1.

### DATE_TRUNC: round down to the start of a period
```sql
SELECT DATE_TRUNC('month', '2025-12-25'::DATE);   -- 2025-12-01
SELECT DATE_TRUNC('year',  '2025-12-25'::DATE);   -- 2025-01-01
SELECT LAST_DAY('2025-02-10'::DATE);              -- 2025-02-28
```
Standard tool for monthly and yearly reports:
```sql
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount)
FROM ORDERS
GROUP BY 1;
```
`DATE_TRUNC('week', ...)` depends on the session `WEEK_START` setting (default Monday).

---

## 3. Practical Date Handling

### Day of week and "next" dates
```sql
SELECT DAYNAME('2025-12-25'::DATE);               -- Thu
SELECT DAYOFWEEKISO('2025-12-25'::DATE);          -- 4 (Monday = 1 ... Sunday = 7)
SELECT NEXT_DAY('2025-12-25'::DATE, 'monday');    -- 2025-12-29
```
Prefer `DAYNAME` or `DAYOFWEEKISO`. Plain `DAYOFWEEK` depends on the `WEEK_START` setting.

### IFF: one-line if/else
```sql
IFF(condition, value_if_true, value_if_false)
SELECT IFF(5 > 3, 'yes', 'no');   -- yes
```
Same as `CASE WHEN condition THEN a ELSE b END`.

### Accurate age calculation
`DATEDIFF('year', dob, CURRENT_DATE())` only subtracts the year numbers, so it gives one year too many when the birthday hasn't happened yet this year.

```sql
SELECT DATEDIFF('year', dob, CURRENT_DATE())
       - IFF(DATEADD('year', DATEDIFF('year', dob, CURRENT_DATE()), dob) > CURRENT_DATE(), 1, 0) AS age
FROM PEOPLE;
```
Example (dob = 2000-12-25, today = 2026-10-10):
1. `DATEDIFF('year', ...)` = 26
2. `DATEADD('year', 26, dob)` = this year's birthday = 2026-12-25
3. Is this year's birthday after today? Yes, so `IFF` returns 1
4. 26 - 1 = **25**

The check compares **this year's birthday** with today (not the date of birth with today).

Simpler alternative:
```sql
SELECT FLOOR(MONTHS_BETWEEN(CURRENT_DATE(), dob) / 12) AS age FROM PEOPLE;
```

### Date spine: every date between two dates
```sql
SELECT DATEADD('day', ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1, '2025-01-01'::DATE) AS d
FROM TABLE(GENERATOR(ROWCOUNT => 31));
```
- `GENERATOR(ROWCOUNT => n)` creates n empty rows
- `SEQ4()` returns a counter, but like sequences it can **skip numbers or come out of order** in parallel runs
- `ROW_NUMBER()` gives clean 1, 2, 3... and `- 1` makes it start from 0
- Use it to find **missing days** by joining your data to this list

### Session format settings
```sql
SHOW PARAMETERS LIKE 'DATE%FORMAT' IN SESSION;
ALTER SESSION SET DATE_OUTPUT_FORMAT = 'DD-MON-YYYY';
```
- `DATE_INPUT_FORMAT` (default `AUTO`): how text dates are guessed
- `DATE_OUTPUT_FORMAT` (default `YYYY-MM-DD`): how dates are displayed
- They change guessing and display only; stored data is the same

### Common mistakes
- Storing dates as `VARCHAR` (breaks sorting and date math); convert with `TO_DATE` on load
- Relying on auto-guessing for `DD/MM` vs `MM/DD` sources
- Comparing a `DATE` with text that has a time part (use timestamps for that)

---

## Quick Lab
```sql
SELECT TO_DATE('25-12-2025', 'DD-MM-YYYY') AS d1,
       TRY_TO_DATE('31-02-2025', 'DD-MM-YYYY') AS d2,
       '2025-12-25'::DATE + 7 AS d3;            -- 2025-12-25, NULL, 2026-01-01

SELECT DATEDIFF('day', '2025-12-01', '2025-12-25') AS days_between,
       DATEADD('month', 1, '2025-01-12'::DATE) AS plus_month,
       DATE_TRUNC('month', '2025-12-25'::DATE) AS month_start;   -- 24, 2025-02-12, 2025-12-01
```

---

## Quick Summary

| Point | Remember |
|---|---|
| DATE | Year, month, day only; displays as `YYYY-MM-DD` |
| TIME | Time of day only (`HH24:MI:SS`), no date, no time zone |
| Text to date | `TO_DATE(text, 'DD-MM-YYYY')`: format describes how to **read the input** |
| Bad values | `TO_DATE` errors, `TRY_TO_DATE` returns NULL and continues |
| Display format | `TO_CHAR(date, 'DD-MON-YYYY')` gives a string |
| Add days | `date + 7`; any unit with `DATEADD(unit, n, date)` |
| Month math | Same day number kept; last valid day used if it doesn't exist |
| DATEDIFF | `(unit, start, end)`; counts boundaries crossed |
| Reports | `DATE_TRUNC('month', date)` for monthly grouping |
| Age | Compare this year's birthday with today, or `FLOOR(MONTHS_BETWEEN(...) / 12)` |
| IFF | `IFF(condition, a, b)`, one-line if/else |
| Date spine | `GENERATOR` + `ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1` |
| Session formats | Change guessing and display, not stored data |
