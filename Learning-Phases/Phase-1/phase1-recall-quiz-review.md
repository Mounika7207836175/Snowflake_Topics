# Phase 1 Recall Quiz — Answers Review

## Topics 1-2: What Snowflake Is / Problems It Solves

**Q1. What does "cloud-native SaaS" mean?**
Her answer: SaaS = software as a service, rent instead of install, pay per use; cloud-native = built to run in the cloud.
✅ Correct.

**Q2. Why is guessing capacity a problem for on-premises warehouses?**
Her answer: Must guess capacity before buying servers; too low -> slows/crashes on peak days (e.g., e-commerce sales day); too high -> paying all year for rarely-used capacity. Snowflake solves this with scale up/scale out.
✅ Correct.

**Q3. Why did storage and compute being bundled together cause slowdowns for concurrent teams?**
Her answer: Skipped.
✔️ Correct answer: Bundled storage+compute means all users share one fixed pool of power. Concurrency (many users querying at once) causes resource contention — teams compete for the same limited power, so one team's heavy query slows another's.

**Q4. Why was JSON difficult for traditional warehouses, and what does Snowflake use to solve it?**
Her answer: Traditional warehouses only understand rows/columns; JSON needed conversion first. Snowflake's VARIANT type stores JSON directly and allows querying it.
✅ Correct.

**Q5. What is a data silo, and how does Snowflake's data sharing solve the stale-copy problem?**
Her answer: Described the stale-copy problem well (CSV via email/SFTP goes out of date; Snowflake sharing gives the consumer live query access, revocable anytime) but didn't define "data silo" itself.
⚠️ Partial. **Data silo** = data locked inside one team's/system's own environment, cut off from other teams, making it hard to combine for cross-team analysis. The sharing explanation (live access, no stale copies, revocable) was correct.

## Topic 3: Cloud DWH vs Traditional Database

**Q6. Difference between "a database hosted on a cloud VM" and "a true cloud-native platform"?**
Her answer: Hosting = creating a VM (e.g., on Azure) and installing database software (SSMS/MySQL) on it, paying for the VM regardless of use. Cloud-native = running queries directly in the cloud platform itself, no software installation, pay-per-use.
✅ Correct.

## Topic 4: Editions

**Q7. What's the one feature that separates Standard from Enterprise edition?**
Her answer: Said Standard can't store JSON (incorrect) and that Enterprise gives more Time Travel (correct but incomplete).
⚠️ Needs correction. Both Standard and Enterprise support JSON via VARIANT — that's not a differentiator. Key differences: Enterprise adds **multi-cluster warehouses** (auto scale-out for concurrency) and **extended Time Travel** (up to 90 days vs 1 day in Standard).

**Q8. Key difference between Business Critical and Virtual Private Snowflake (VPS)?**
Her answer: Business Critical and lower editions share the same underlying Snowflake infrastructure pool (logically separated, not physically); Business Critical adds stronger network security; VPS has its own completely separate, dedicated infrastructure and is very costly, used rarely.
✅ Correct (core distinction: logical vs physical separation).

## Topic 5: Cloud Providers and Regions

**Q9. Why can't you switch a Snowflake account's cloud provider later?**
Her answer: The cloud provider is chosen at account creation; switching later would mean physically moving data from one cloud to another — a huge, complex process. The workaround is creating a new account on the target provider and migrating/cloning data across.
✅ Correct.

**Q10. Name two reasons region choice matters.**
Her answer: Latency (region should be close to users) — gave correct reasoning with a North America/South America example. Second reason not recalled.
⚠️ Partial. Second reason: **data residency/compliance** — some countries legally require certain data to stay stored within their borders.

## Topic 6: Organizations and Accounts

**Q11. Difference between an account and an organization?**
Her answer: An account is for an individual/one team; an organization sits on top, grouping multiple accounts together.
⚠️ Mostly correct, one imprecision: an account isn't limited to one person/team — it's a full workspace that can serve an entire company. An organization groups multiple such accounts for centralized administration/billing.

**Q12. One real reason for separate DEV and PROD accounts instead of just separate databases?**
Her answer: Separate accounts mean separate compute power, so DEV's workload can't affect PROD's performance, and vice versa.
✅ Correct.

## Topic 9: Object Hierarchy

**Q13. Why do Warehouses, Users, and Roles sit outside the Database → Schema hierarchy?**
Her answer: They aren't related to the actual data being stored (like sales data) — they're about the Snowflake account itself.
✅ Correct.

**Q14. Why don't two tables named `CUSTOMERS` in two different databases conflict?**
Her answer: Snowflake uses fully qualified names (database.schema.table), and you select the database/schema context before querying.
✅ Correct.

**Q15. Core difference between a Table and a View?**
Her answer: A table stores actual data; a view doesn't store data, only the query definition, and fetches data live when queried.
✅ Correct.

**Q16. Difference between a Stored Procedure and a UDF?**
Her answer: Said a Stored Procedure "should not return any value," while a UDF can return a value and be used inside a SELECT statement.
⚠️ Needs correction. A Stored Procedure *can* return a single value — the real distinction is that a **UDF** is usable *inline* inside a SELECT/expression (returns a SQL-usable value), while a **Stored Procedure** is called on its own (not inside a SELECT), can run multiple steps, and can include DDL (like creating tables).

## Topic 10: Sessions

**Q17. Why can two worksheets have completely different active roles/databases at the same time?**
Her answer: Said it's to make it easier to maintain a session and avoid confusion when working with different databases.
⚠️ Close but not the core reason. Each worksheet opens its **own separate connection/session** to Snowflake — contexts are independent by design (how sessions work), not a choice made for convenience.

**Q18. What happens to a session variable (`SET`) when the session ends?**
Her answer: It closes/disappears when the session ends.
✅ Correct.

## Topic 11: SQL Dialect

**Q19. What does the `:` operator let you do, and on what data type does it work?**
Her answer: Retrieves fields from JSON-format data stored in a VARIANT column.
✅ Correct.

**Q20. Advantage of `QUALIFY` over wrapping a query in a subquery with `ROW_NUMBER()`?**
Her answer: `QUALIFY` lets you apply a condition on a window function's result column directly, without needing a subquery.
✅ Correct.

## Topic 12: Metadata and System Functions

**Q21. Difference between `INFORMATION_SCHEMA` and `SNOWFLAKE.ACCOUNT_USAGE`?**
Her answer: Skipped (flagged as a weak topic).
✔️ Correct answer: `INFORMATION_SCHEMA` = live, per-database, current-state metadata only. `SNOWFLAKE.ACCOUNT_USAGE` = account-wide, historical (up to ~1 year), and includes objects that have since been deleted.

**Q22. Which one would you use to find out which warehouse consumed the most credits last month?**
Her answer: Skipped.
✔️ Correct answer: `SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` — `INFORMATION_SCHEMA` has no historical usage trends.

---

## Summary — Areas to Revisit
- Standard vs Enterprise edition features (Q7)
- Data silo definition (Q5)
- Second reason region choice matters — compliance/data residency (Q10)
- Stored Procedure vs UDF distinction (Q16)
- Why worksheet sessions are independent (Q17)
- Metadata and system functions topic overall (Q21, Q22) — worth a full re-read of the Topic 12 notes file
