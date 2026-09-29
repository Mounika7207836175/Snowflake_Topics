# Phase 1 | Topic 7: Snowsight Web Interface

**Snowsight** is Snowflake's web-based interface (the website you log into). It replaced the older, more basic "Classic Console."

## Main Navigation Areas (Left Sidebar)
1. **Home** — landing page: recent worksheets, announcements, quick links.
2. **Worksheets / Workspace** — where you write and run SQL.
3. **Databases** — browse the object hierarchy visually (databases → schemas → tables/views/stages) instead of only querying `INFORMATION_SCHEMA`.
4. **Data Products / Marketplace** — browse datasets other companies/providers share publicly or privately (ties to the data sharing concept from Topic 2).
5. **Activity / Query History** — every query run: execution time, success/failure, warehouse used. Key tool for debugging.
6. **Admin section** — manages:
   - **Warehouses** (create, resize, monitor usage)
   - **Users & Roles** (login access, permissions)
   - **Billing/Usage** (credits consumed)
   - **Security** (network policies, sessions)

## Extra Details
- **Context switcher (top of worksheet):** shows current **role, warehouse, database, schema** — the visual version of `USE ROLE / USE WAREHOUSE / USE DATABASE / USE SCHEMA` (session context). Switch any of these by clicking, no SQL needed.
- **Notebooks (newer feature):** mix SQL, Python, and markdown in one document (like Jupyter) — useful for data science exploration. Worksheets remain pure SQL. Not commonly required for a Data Engineer role, but good to recognize.
- **Dashboards:** pin query results as charts/tiles for quick visual monitoring, without needing an external BI tool for simple cases.

## Interview Angles
- *"If a query is running slowly, where would you investigate in Snowsight?"* — **Query History / Activity** tab, which lets you open a **Query Profile** (visual breakdown of where time was spent) — covered more in later performance topics.
- *"How would you check which role/warehouse a query used?"* — The context switcher bar while writing it live, or **Query History** afterward, which logs role/warehouse for every past query.

---
**Next Topic:** Worksheets and workspaces
