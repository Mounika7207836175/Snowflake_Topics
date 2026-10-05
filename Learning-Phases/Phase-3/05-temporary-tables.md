# Phase 3 | Topic 5: Temporary Tables

## What Makes a Table "Temporary"
The third and most restrictive Standard sub-type — for data only needed for a short-lived purpose within a single working session.

**Key defining behavior:** a Temporary table **only exists for the duration of the session that created it.** The moment that session ends (logout, closed connection, timeout), Snowflake **automatically and permanently deletes** the table — no `DROP` needed, no recovery possible.

## Full Restrictions (vs Permanent and Transient)
1. **Time Travel:** maximum 1 day (same ceiling as Transient) — largely irrelevant since the table won't outlive the session anyway.
2. **Fail-safe:** zero, same as Transient.
3. **Visibility:** only visible to the **session that created it**. A second worksheet (a separate session) cannot see or query it, even for the same logged-in user.
4. **Naming collision / "shadowing" (a real, tricky detail):** creating a Temporary table with the **same name** as an existing Permanent table in the same schema causes the session to **transparently use the Temporary one** for all queries — silently shadowing the Permanent table, with no error or warning. A genuine source of confusing bugs.

## Why Temporary Tables Exist
Useful for **scratch work within a single script or session** — holding intermediate calculation results without cluttering the database permanently, with automatic cleanup when the session ends.

**Interview angle:** *"If you create a Temporary table in one worksheet, can you see it from a second worksheet as the same user?"*
No — Temporary tables are scoped to the exact session that created them; a different worksheet is a different session.

## Hands-On: Proving Session-Scoping and Shadowing

```sql
-- Step 1 (original worksheet): create and query
CREATE TEMPORARY TABLE SALES_DB.RAW.SCRATCH_CALC (
    calc_id STRING,
    result_value STRING
);
INSERT INTO SALES_DB.RAW.SCRATCH_CALC VALUES ('CALC1', '999');
SELECT * FROM SALES_DB.RAW.SCRATCH_CALC;   -- returns the row successfully

-- Step 2 (NEW worksheet / separate session):
SELECT * FROM SALES_DB.RAW.SCRATCH_CALC;   -- fails: "does not exist"

-- Step 3 (back in original worksheet): confirm type
SELECT GET_DDL('TABLE', 'SALES_DB.RAW.SCRATCH_CALC');  -- shows CREATE OR REPLACE TEMPORARY TABLE
```
**Result confirmed:** the second worksheet genuinely could not find the table — proving session-scoping directly.

## Recall Q&A

**Scenario:** A Temporary table named `ORDERS_RAW` (same name as the real Permanent table) gets created in a worksheet without realizing it. Running `SELECT * FROM ORDERS_RAW` in that worksheet returns unexpected/wrong data. What's happening, and how would you debug it?

**Answer:** The Temporary table silently shadows the Permanent one — the query hits the temp table instead, with no warning. To debug: check `GET_DDL('TABLE', 'ORDERS_RAW')` or examine the object's type directly, to reveal the `TEMPORARY` keyword exposing the shadow, rather than assuming the underlying real data itself is wrong. In practice, suspiciously unexpected data is usually the first clue prompting this check.

**Interview angle:** *"How would you debug unexpectedly wrong data from a table you're certain should be correct?"*
Check if a same-named Temporary table might be shadowing it in the current session, using `GET_DDL` or checking the object's type, rather than assuming the underlying data is wrong.

---
**Next Topic:** Dynamic tables
