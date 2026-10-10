# Snowflake: Tags and Policies (Complete Notes)

Both belong to **data governance**: knowing what data you have and controlling who sees what.

- **Tags** = **labels** attached to objects. They describe things.
- **Policies** = **rules** attached to objects. They enforce things.

| | Tag | Policy |
|---|---|---|
| Job | Label and classify | Protect and restrict |
| Example | "This column is PII" | "Only HR can see the real salary" |
| Changes query results? | No | Yes |

**Policy types at a glance**

| Policy | Controls | Works on |
|---|---|---|
| Masking policy | Which **value** is shown in a column | Columns |
| Row access policy | Which **rows** a role can see | Tables, views |
| Aggregation / projection policy | How data may be queried (group size, column output) | Tables |
| Network policy | Which **IP addresses** can connect | Account, users |
| Password / session policy | Password rules, session timeouts | Account, users |

---

## 1. Tags

A **tag** is a schema-level object: a key-value label you can attach to databases, schemas, tables, columns, warehouses, users and more.

**Step 1: create the tag**
```sql
CREATE TAG PII_TAG;

-- limit allowed values
CREATE TAG SENSITIVITY ALLOWED_VALUES 'public', 'internal', 'confidential';
```

**Step 2: attach it to an object**
```sql
ALTER TABLE CUSTOMERS MODIFY COLUMN email SET TAG PII_TAG = 'email';
ALTER TABLE CUSTOMERS SET TAG SENSITIVITY = 'confidential';
```
The tag is the label; `'email'` or `'confidential'` is the **value**.

**Step 3: find what is tagged**
```sql
SELECT SYSTEM$GET_TAG('PII_TAG', 'CUSTOMERS.EMAIL', 'COLUMN');

SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_TAG';
```
The second query lists **every object** carrying that tag ("where is all our sensitive data?").

**Tag inheritance**
- Tag a **table** and its **columns inherit** the tag automatically
- Tag a **schema** and everything inside it inherits
- You can override the value on a lower object
- Inheritance is automatic and involves **no policy**

**Uses of tags**
- **Classification**: mark PII or confidential columns
- **Cost tracking**: tag warehouses by team or project and report credits per tag
- **Governance reporting**: audit which objects hold sensitive data
- **Tag-based masking**: attach a masking policy to a tag (see section 4)

**Manage**
```sql
SHOW TAGS;
ALTER TABLE CUSTOMERS MODIFY COLUMN email UNSET TAG PII_TAG;
DROP TAG PII_TAG;
```

---

## 2. Masking Policies

A **masking policy** hides or changes a column's value **at query time**, depending on who is asking. The stored data is **never changed**.

**Example: real email for HR, masked for everyone else**
```sql
-- Step 1: create the policy
CREATE OR REPLACE MASKING POLICY EMAIL_MASK
AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('HR_ROLE') THEN val
    ELSE '*****'
  END;

-- Step 2: attach it to a column
ALTER TABLE CUSTOMERS MODIFY COLUMN email
  SET MASKING POLICY EMAIL_MASK;

-- Step 3: see the effect
USE ROLE HR_ROLE;
SELECT email FROM CUSTOMERS;      -- raj@gmail.com

USE ROLE ANALYST_ROLE;
SELECT email FROM CUSTOMERS;      -- *****
```
- `(val STRING)`: the column's original value
- `RETURNS STRING`: must be the **same data type** as the column
- The `CASE` decides what each role sees

**Partial masking**
```sql
CASE
  WHEN CURRENT_ROLE() = 'HR_ROLE' THEN val
  ELSE REGEXP_REPLACE(val, '.+@', '*****@')
END
```
Result for other roles: `*****@gmail.com`.

**Key points**
- One policy can be **reused on many columns** (same data type)
- A column can have **only one** masking policy at a time
- Stored data stays unchanged: no copy, no duplicate table
- Masking policies need **Enterprise edition** or higher

**Manage**
```sql
SHOW MASKING POLICIES;
ALTER TABLE CUSTOMERS MODIFY COLUMN email UNSET MASKING POLICY;
DROP MASKING POLICY EMAIL_MASK;   -- first unset it from all columns
```

**Lab idea:** create `HR_ROLE` and `ANALYST_ROLE` like in the stored procedure lab, apply the policy, and switch roles to compare results.

---

## 3. Row Access Policies

A **row access policy** decides **which rows** a role can see. Rows that fail the rule are simply **not returned**, as if they didn't exist.

- Masking policy = same rows, **different values**
- Row access policy = **different rows**

**Example: each regional role sees only its own region**
```sql
-- Step 1: create the policy (must return BOOLEAN)
CREATE OR REPLACE ROW ACCESS POLICY REGION_POLICY
AS (region_val STRING) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() = 'ADMIN_ROLE' THEN TRUE
    WHEN CURRENT_ROLE() = 'SALES_INDIA' AND region_val = 'INDIA' THEN TRUE
    WHEN CURRENT_ROLE() = 'SALES_US'    AND region_val = 'US'    THEN TRUE
    ELSE FALSE
  END;

-- Step 2: attach it to the table, naming the column(s) it reads
ALTER TABLE SALES ADD ROW ACCESS POLICY REGION_POLICY ON (region);

-- Step 3: test
USE ROLE SALES_INDIA;
SELECT * FROM SALES;     -- only INDIA rows
```
- `TRUE` = the row is visible, `FALSE` = the row is hidden
- The `ON (region)` part tells Snowflake which column to pass into the policy

**Key points**
- A table or view can have **one** row access policy at a time
- It applies to `SELECT`, and also to `UPDATE`/`DELETE` (a role can only change rows it can see)
- It can be used together with masking policies on the same table
- Needs **Enterprise edition** or higher
- For many roles, a common design is a **mapping table** (role -> allowed region) that the policy looks up, instead of hardcoding every role

**Manage**
```sql
SHOW ROW ACCESS POLICIES;
ALTER TABLE SALES DROP ROW ACCESS POLICY REGION_POLICY;
DROP ROW ACCESS POLICY REGION_POLICY;
```

---

## 4. Tag-Based Masking

Instead of attaching a masking policy to each column by hand, you attach it **to a tag**. Every column carrying that tag is then masked automatically.

```sql
-- 1. A masking policy
CREATE OR REPLACE MASKING POLICY EMAIL_MASK
AS (val STRING) RETURNS STRING ->
  CASE WHEN CURRENT_ROLE() = 'HR_ROLE' THEN val ELSE '*****' END;

-- 2. A tag
CREATE TAG PII_TAG;

-- 3. Attach the policy to the tag
ALTER TAG PII_TAG SET MASKING POLICY EMAIL_MASK;

-- 4. Tag any column; it is now masked automatically
ALTER TABLE CUSTOMERS MODIFY COLUMN email SET TAG PII_TAG = 'email';
ALTER TABLE ORDERS    MODIFY COLUMN cust_email SET TAG PII_TAG = 'email';
```

**Key points**
- One tag can have **one masking policy per data type** (for example one for STRING columns, one for NUMBER)
- Because of **tag inheritance**, tagging a table or schema can protect many columns at once
- A policy set **directly on a column** takes precedence over a tag-based one
- Tag the data once and protection follows; no need to remember every column

**Remove**
```sql
ALTER TAG PII_TAG UNSET MASKING POLICY EMAIL_MASK;
```

---

## 5. Other Policy Types (overview only)

- **Aggregation policy**: forces queries on a table to return **groups of at least N rows**, so individuals can't be singled out
- **Projection policy**: stops a column from being **shown in query output**, though it may still be used in filters or joins
- **Network policy**: allow or block **IP addresses** at account or user level
  ```sql
  CREATE NETWORK POLICY OFFICE_ONLY ALLOWED_IP_LIST = ('203.0.113.0/24');
  ALTER USER some_user SET NETWORK_POLICY = OFFICE_ONLY;
  ```
- **Password policy**: rules for length, complexity, and expiry
- **Session policy**: idle timeout settings for sessions

Know what each one is for. The deep dives are the masking, row access and tag-based ones above.

---

## 6. Privileges and Viewing (quick reference)

- Creating and attaching these objects needs privileges such as `CREATE MASKING POLICY`, `CREATE ROW ACCESS POLICY`, `CREATE TAG`, and the `APPLY` privileges (for example `APPLY MASKING POLICY`, `APPLY TAG`) granted to a role
- Policy creation is usually handled by a **governance/admin role**, separate from the people who own the tables
- To see where policies are applied, use `SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES` or the `POLICY_REFERENCES` table function

---

## Quick Summary

| Point | Remember |
|---|---|
| Tag | A label (key-value) on an object; describes data |
| Policy | A rule on an object; controls access |
| Tag inheritance | Table tag flows to its columns, schema tag flows to everything inside; no policy involved |
| Find tagged objects | `SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES` |
| Masking policy | Changes what a role **sees** in a column; stored data unchanged |
| Must match | Masking policy return type = column data type |
| Masking limit | One masking policy per column; reusable across columns |
| Row access policy | Returns BOOLEAN per row; `FALSE` rows are hidden |
| Row access limit | One row access policy per table or view |
| Masking vs row access | Masking = different values, row access = different rows |
| Tag-based masking | `ALTER TAG ... SET MASKING POLICY`; one policy per data type per tag |
| Precedence | Policy set directly on a column beats the tag-based one |
| Editions | Masking and row access need Enterprise or higher |
| Manage | `SHOW`, `UNSET` / `DROP ... POLICY` from the object, then drop the policy |
