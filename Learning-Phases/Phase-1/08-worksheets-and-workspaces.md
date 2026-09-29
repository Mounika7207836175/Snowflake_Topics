# Phase 1 | Topic 8: Worksheets and Workspaces

## What a Worksheet Is
A **Worksheet** is a single SQL document — one file to write and run queries, similar to a `.sql` file. Each worksheet:
- Has its own **saved session context** (role, warehouse, database, schema) — different worksheets can work against completely different databases at the same time, side by side.
- Keeps a **history** of what was run and the results, until cleared/closed.
- Can be **renamed and organized**.

**Workspace ("My Workspace")** = the folder/container holding all worksheets, like a file explorer for SQL files. Worksheets can be organized into folders within it.

**Sharing worksheets:** you can share a specific worksheet with a teammate — they see the *code*, but their ability to run it depends on their own role's permissions. This is different from **data sharing** (Topic 2), which shares underlying data access, not code.

**Interview angle:** *"If two worksheets are open, querying two different databases at once, will they interfere?"*
No — each worksheet has its own independent session context, so they're fully isolated even in the same browser tab.

## Git Integration
Snowsight can link a worksheet (or folder of worksheets) to a **Git repository** (e.g., GitHub):
- Connect Snowsight to a GitHub repo, edit/run `.sql` files stored there directly inside Snowsight, and commit changes back.
- Matters because real-world SQL code is almost always **version-controlled**, not just saved loosely — same idea as manually pushing `.md` notes to a personal repo.

**Interview angle:** *"Why is version control important for SQL/data engineering code?"*
Tracks who changed what and when, allows rolling back mistakes, supports code review before production changes, and lets multiple engineers collaborate without overwriting each other's work.

---
**Next Topic:** Snowflake object hierarchy
