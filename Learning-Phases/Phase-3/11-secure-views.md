# Snowflake secure views

## Core concept

A non-materialized secure view stores a SELECT definition rather than a separate dataset. It adds privacy protections: ordinary consumers cannot inspect its definition, and Snowflake restricts optimizations that could indirectly expose filtered-out data. Owners and certain privileged roles can still inspect definitions. Secure views can run more slowly than standard views.

Use them for privacy-sensitive access. For simple query reuse without privacy requirements, standard views are usually sufficient.

Reference: [Snowflake secure views](https://docs.snowflake.com/en/user-guide/views-secure).

## What SECURE does not do

- It does not automatically identify sensitive columns, mask salaries, or choose which rows to hide.
- It does not revoke existing source-table permissions.
- It does not create a stored copy or require a separate refresh.
- It does not make SELECT * safe for every audience: selected columns remain visible.

You choose the output columns and filters. Grants determine who can query the view. An ordinary view can also select fewer columns; SECURE supplies additional protections beyond that selection.

## Hands-on: employee directory without salary

Use an existing warehouse and a role permitted to create these objects. Run CREATE and INSERT statements once; repeating the inserts adds duplicate records. These are fictional employees.

```sql
-- Keep the demonstration objects together for easy inspection.
USE DATABASE LEARNING_DB;
CREATE SCHEMA IF NOT EXISTS SECURE_VIEW_LAB;
USE SCHEMA SECURE_VIEW_LAB;

-- Include a salary column so we can explicitly exclude it from the view.
CREATE TABLE employees (
  employee_id INT,
  employee_name VARCHAR,
  department VARCHAR,
  salary NUMBER(10,2)
);

-- Populate the source with two fictional records.
INSERT INTO employees VALUES
  (1, 'Anita', 'Engineering', 80000),
  (2, 'Ravi', 'Finance', 75000);

-- Select only directory fields; the SELECT list is what excludes salary.
-- SECURE additionally protects the definition and limits unsafe optimizations.
CREATE SECURE VIEW employee_directory_v AS
SELECT employee_id, employee_name, department
FROM LEARNING_DB.SECURE_VIEW_LAB.employees;

-- Expected: two rows with ID, name, and department; no salary column.
SELECT * FROM employee_directory_v;

-- Inspect the object's secure flag rather than assuming the keyword worked.
-- Expected: EMPLOYEE_DIRECTORY_V with IS_SECURE = YES.
SELECT table_name, is_secure
FROM LEARNING_DB.INFORMATION_SCHEMA.VIEWS
WHERE table_schema = 'SECURE_VIEW_LAB'
  AND table_name = 'EMPLOYEE_DIRECTORY_V';
```

## Existing views

```sql
-- Syntax reference only: replace the example name with a real owned view.
-- Add privacy protection without rewriting its SELECT definition.
-- ALTER VIEW existing_view_name SET SECURE;
```

Reference: [CREATE VIEW](https://docs.snowflake.com/en/sql-reference/sql/create-view).

## Permissions and verification

A consumer requires database/schema USAGE and SELECT on the view, plus warehouse access for query execution. Direct SELECT on the underlying employees table is not required merely to consume the view. The view owner must retain the necessary source privileges.

The owning role can see its definition, so querying as the owner does not test definition hiding. A separate consumer-role test is needed to demonstrate that behavior. Check inherited and secondary-role permissions before concluding a consumer is isolated from the source.

The lab verifies output columns and the secure flag. It does not demonstrate every privacy protection or a separate consumer-role permission boundary.

## Production example

An employee directory should expose name and department, while compensation reporting has a different audience. Give directory users access to the directory view without granting them access to the full source. If a user's other role already permits reading salaries from the source, this view does not override that permission.

## Review answers

1. **Does SELECT * inside a secure view automatically hide salary?** No. Salary is included in the selected output.
2. **Does a secure view block existing direct source-table access?** No. Source grants must be managed separately.

## Status

Learner confirmed understanding/results in chat. A separate restricted-consumer-role test has not been performed. Next roadmap topic: materialized views.
