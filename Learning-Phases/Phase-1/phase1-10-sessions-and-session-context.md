# Phase 1 | Topic 10: Sessions and Session Context

## What a Session Is
A **session** starts the moment you connect/log in and stays alive until you log out, disconnect, or it **times out** from inactivity. Every worksheet, BI tool connection, or script connecting to Snowflake creates its **own separate session**.

**What a session holds:**
- Current **role**
- Current **warehouse**
- Current **database** and **schema**
- Any **temporary objects** created (temp tables, session variables) — gone when session ends
- Session-level settings (date formats, timezone)

**Why it matters:** each worksheet is its own independent connection/session — one can be `ANALYST_ROLE` on `SALES_DB` while another is `SYSADMIN_ROLE` on `HR_DB`, at the same time, with zero interference. (Note: this isolation is because each worksheet opens a separate connection — not because of fully qualified names. Fully qualified names instead let you reference an object *outside* your current session's database/schema without switching context.)

**Interview angle:** *"If a session times out mid-query, what happens?"* The running query is typically cancelled; reconnecting creates a new session, and any temp tables/session variables from the old one are lost.

## Viewing and Controlling Session Context
Check current context:
```sql
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

Change it:
```sql
USE ROLE SYSADMIN;
USE WAREHOUSE etl_wh;
USE DATABASE SALES_DB;
USE SCHEMA ANALYTICS;
```

## Session Variables
Store a temporary value for the session's duration, reusable across queries:
```sql
SET my_threshold = 1000;
SELECT * FROM orders WHERE amount > $my_threshold;
```
Disappears when the session ends — not saved permanently.

## Session Timeout
Default inactivity timeout is commonly ~4 hours, configurable by an account admin via **session policies**. After timeout, reconnecting is required.

**Interview angle:** *"How would you pass a dynamic value into multiple queries without hardcoding it each time?"* Session variables (`SET`) — set once, reference with `$variable_name` across queries in that session.

---
**Next Topic:** Snowflake SQL dialect
