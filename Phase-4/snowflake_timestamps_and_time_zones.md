# Snowflake: Timestamp Types and Time Zones

A **timestamp** stores a **date and a time together**. Snowflake has three kinds, and the only difference between them is **how they treat time zones**.

---

## 1. The Three Timestamp Types

### Plain-words picture
The same moment shows on different clocks in different countries. When it's 10:00 AM in India, it's 11:30 PM (the previous day) in New York.

- **TIMESTAMP_NTZ** is a **sticky note**: "10:00 on 25 Dec". No country's clock attached. Stored as written, never changes.
- **TIMESTAMP_LTZ** is a **real moment**: Snowflake remembers the actual moment (UTC inside) and shows it on the **reader's** (session's) clock.
- **TIMESTAMP_TZ** is a **real moment with a label**: it keeps its own offset (like `+0530`) and does not shift with the session.

| Type | Stores | Display |
|---|---|---|
| `TIMESTAMP_NTZ` | Date + time, no zone | Always the same |
| `TIMESTAMP_LTZ` | A moment (UTC inside) | Shifts with the session time zone |
| `TIMESTAMP_TZ` | A moment + its own offset | Shows that offset |

```sql
SELECT '2025-12-25 10:00:00'::TIMESTAMP_NTZ;        -- 2025-12-25 10:00:00.000
SELECT '2025-12-25 10:00:00'::TIMESTAMP_LTZ;        -- read in the session's time zone
SELECT '2025-12-25 10:00:00 +0530'::TIMESTAMP_TZ;   -- 2025-12-25 10:00:00.000 +0530
SELECT '2025-12-25 10:00:00 -0500'::TIMESTAMP_TZ;   -- each value keeps its own tag
```

### Plain `TIMESTAMP`
A name for one of the three. By default it means `TIMESTAMP_NTZ` (controlled by `TIMESTAMP_TYPE_MAPPING`). Many teams write the full type name for clarity.

### Which to use
- **NTZ**: time zone doesn't matter or is already handled ("store opens at 9:00"), or source data has no zone
- **LTZ**: one real moment that should display correctly for users in different places
- **TZ**: data from several regions where you want to keep each original offset

### See the difference yourself
```sql
ALTER SESSION SET TIMEZONE = 'Asia/Kolkata';

CREATE OR REPLACE TABLE TS_DEMO (ntz TIMESTAMP_NTZ, ltz TIMESTAMP_LTZ);
INSERT INTO TS_DEMO SELECT '2025-12-25 10:00:00', '2025-12-25 10:00:00';
SELECT * FROM TS_DEMO;
-- ntz: 2025-12-25 10:00:00   ltz: 2025-12-25 10:00:00 +0530

ALTER SESSION SET TIMEZONE = 'America/New_York';
SELECT * FROM TS_DEMO;
-- ntz: 2025-12-25 10:00:00   ltz: 2025-12-24 23:30:00 -0500
```
- `ntz` stays the same because it is only a note with no zone to convert
- `ltz` shifts because it is the same moment, now read on a New York clock

### Precision
All three accept 0 to 9 fractional-second digits, for example `TIMESTAMP_NTZ(3)` for milliseconds. Default is 9 (nanoseconds).

### "Now" functions
```sql
SELECT CURRENT_TIMESTAMP();   -- TIMESTAMP_LTZ, in the session time zone
SELECT SYSDATE();             -- TIMESTAMP_NTZ, always in UTC
```

---

## 2. Converting and Common Traps

### CONVERT_TIMEZONE
```sql
SELECT CONVERT_TIMEZONE('Asia/Kolkata', 'America/New_York',
                        '2025-12-25 10:00:00'::TIMESTAMP_NTZ);
-- 2025-12-24 23:30:00.000
```
Read as: "this is an India clock reading, give me the New York clock reading."

Short form (target zone first, then a timestamp that already has a zone):
```sql
SELECT CONVERT_TIMEZONE('America/New_York', '2025-12-25 10:00:00 +0530'::TIMESTAMP_TZ);
-- 2025-12-24 23:30:00.000 -0500
```
Use zone **names** like `'Asia/Kolkata'`, not fixed offsets, because names handle **daylight saving** automatically. "Asian time zone" is not a valid zone; always use the exact name.

### Text to timestamp, with a format
```sql
SELECT TO_TIMESTAMP_NTZ('25-12-2025 14:30:00', 'DD-MM-YYYY HH24:MI:SS');
-- 2025-12-25 14:30:00.000

SELECT TRY_TO_TIMESTAMP_NTZ('bad value', 'DD-MM-YYYY HH24:MI:SS');   -- NULL
```
`TRY_` returns NULL for bad values instead of failing.

### Timestamp to date, and the midnight trap
```sql
SELECT '2025-12-25 14:30:00'::TIMESTAMP_NTZ::DATE;   -- 2025-12-25
```

The common bug is comparing a timestamp with a plain date. The text `'2025-12-25'` becomes `2025-12-25 00:00:00`, so only rows at **exactly midnight** match (this happens even with plain NTZ; it is not a time zone problem).
```sql
-- WRONG: matches only 00:00:00
SELECT * FROM EVENTS WHERE event_ts = '2025-12-25';

-- RIGHT (option 1): cast to date
SELECT * FROM EVENTS WHERE event_ts::DATE = '2025-12-25';

-- RIGHT (option 2): a range, usually faster on big tables
SELECT * FROM EVENTS
WHERE event_ts >= '2025-12-25' AND event_ts < '2025-12-26';
```
Use `>=` and `<` (not `<=` with the next day, which would include midnight of the next day).

### Session clock
```sql
SHOW PARAMETERS LIKE 'TIMEZONE' IN SESSION;
ALTER SESSION SET TIMEZONE = 'UTC';
```
Accounts usually default to `America/Los_Angeles`. This setting only affects how **LTZ** values are read and shown (and what `CURRENT_TIMESTAMP()` shows).

---

## 3. Timestamps in Practice

### Date math with timestamps
```sql
SELECT DATEADD('hour', 5, '2025-12-25 10:00:00'::TIMESTAMP_NTZ);       -- 2025-12-25 15:00:00.000
SELECT DATEDIFF('minute', '2025-12-25 10:00:00'::TIMESTAMP_NTZ,
                          '2025-12-25 10:45:00'::TIMESTAMP_NTZ);       -- 45
SELECT DATE_TRUNC('hour', '2025-12-25 10:45:30'::TIMESTAMP_NTZ);       -- 2025-12-25 10:00:00.000
SELECT HOUR('2025-12-25 10:45:30'::TIMESTAMP_NTZ);                     -- 10
```
Same functions as for dates, with smaller units (`hour`, `minute`, `second`). `DATEDIFF` still counts **boundaries crossed**: 10:59 to 11:01 is 2 minutes apart, but `DATEDIFF('hour', ...)` returns 1.

### Epoch numbers
Many systems store time as seconds since 1 Jan 1970 (UTC).
```sql
SELECT TO_TIMESTAMP(0);              -- 1970-01-01 00:00:00.000 (seconds)
SELECT TO_TIMESTAMP(1000, 3);        -- 1970-01-01 00:00:01.000 (3 = milliseconds)
SELECT DATE_PART(EPOCH_SECOND, '1970-01-01 00:01:00'::TIMESTAMP_NTZ);   -- 60
```
- Second argument is the **scale**: 0 = seconds, 3 = milliseconds, 9 = nanoseconds
- A time that looks like it's from the year 50000 usually means milliseconds were passed without the scale

### Loading data with no time zone
If a source has `2025-12-25 10:00:00` with no zone, Snowflake can't know whose clock it was. Ask the source team which zone they use, then convert.
```sql
SELECT CONVERT_TIMEZONE('Asia/Kolkata', 'UTC', source_ts::TIMESTAMP_NTZ) AS created_at_utc
FROM RAW_EVENTS;
```
Source on India time means read it as `Asia/Kolkata`, then convert to UTC.

### Best practice: store in UTC
- Keep the stored column in **UTC** and name it clearly (`created_at_utc`)
- Convert to local time only at **reporting** time:
```sql
SELECT CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', created_at_utc) AS created_at_ist
FROM EVENTS;
```
- This avoids mixed-zone bugs when data arrives from many regions

### Daylight saving traps
- Use zone names (`'America/New_York'`), never fixed offsets like `-05:00`
- A local day can have 23 or 25 hours when clocks change, so "add 24 hours" and "add 1 day" can land on different local times
- Another reason to do calculations in UTC

### Checklist when a timestamp looks wrong
1. Which of the three types is the column?
2. What is the session time zone (`SHOW PARAMETERS LIKE 'TIMEZONE' IN SESSION`)?
3. Was the source in UTC or local time?
4. Is a date being compared with a timestamp (the midnight trap)?

---

## Quick Lab
```sql
SELECT CONVERT_TIMEZONE('Asia/Kolkata', 'UTC', '2025-12-25 10:00:00'::TIMESTAMP_NTZ);
-- 2025-12-25 04:30:00.000 (India is 5h 30m ahead of UTC)

SELECT TO_TIMESTAMP(1000, 3) AS from_ms,
       DATEDIFF('hour', '2025-12-25 10:59:00'::TIMESTAMP_NTZ,
                        '2025-12-25 11:01:00'::TIMESTAMP_NTZ) AS hour_diff;
-- 1970-01-01 00:00:01.000 and 1
```

---

## Quick Summary

| Point | Remember |
|---|---|
| TIMESTAMP_NTZ | Date + time exactly as written, no zone; never changes |
| TIMESTAMP_LTZ | A real moment (UTC inside); shown on the session's clock |
| TIMESTAMP_TZ | A real moment with its own offset; each value keeps its tag |
| TIMESTAMP | A name for one of the three, `NTZ` by default |
| Precision | 0 to 9 fractional digits, default 9 |
| Now | `CURRENT_TIMESTAMP()` returns LTZ; `SYSDATE()` returns NTZ in UTC |
| CONVERT_TIMEZONE | `(from_zone, to_zone, ts)` or `(to_zone, ts_with_zone)`; use zone names |
| Text to timestamp | `TO_TIMESTAMP_NTZ(text, format)`; `TRY_` version returns NULL on bad values |
| Midnight trap | `ts = '2025-12-25'` only matches 00:00:00; use `ts >= day AND ts < next_day` |
| Epoch | `TO_TIMESTAMP(n, scale)`: 0 = seconds, 3 = milliseconds |
| No-zone sources | Ask which zone the source used, then convert |
| Best practice | Store in UTC, convert to local time only when reporting |
| Daylight saving | Zone names handle it; a local day can have 23 or 25 hours |
