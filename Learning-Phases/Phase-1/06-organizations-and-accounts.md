# Phase 1 | Topic 6: Organizations and Accounts

## What an Account Is
An **account** is the actual workspace you log into and use — where databases, warehouses, and data live (what you created at signup).

Every account:
- Belongs to **one specific cloud provider and region** (can't be changed later — see Topic 5).
- Has its own **edition** (Standard, Enterprise, etc.).
- Has its own separate storage, compute, users, and security settings — fully isolated from any other account.

Think of an account as one complete "instance" of Snowflake, dedicated to one purpose.

## Why Companies Use Multiple Accounts
1. **Environment separation:** `DEV` account for testing, `PROD` account for real data — mistakes in DEV can't touch PROD.
2. **Different regions:** separate accounts per region for latency/compliance.
3. **Different business units:** each department (finance, marketing) may get its own account for isolation and independent billing.
4. **Different cloud providers:** separate accounts if parts of the company use AWS vs Azure.

**Interview angle:** *"Why DEV/PROD accounts instead of separate databases in one account?"*
Full account separation = stronger isolation — a runaway query, mistake, or security issue in DEV cannot touch PROD, since no underlying resources are shared at all (unlike separate databases within the same account).

## What an Organization Is
An **Organization** sits above individual accounts — it groups a company's multiple Snowflake accounts (across regions/clouds/environments) under one umbrella for centralized administration.

**What it enables**
- Centralized **visibility** into billing/usage across all accounts (each account is still tracked separately underneath — it's not merged into one bill).
- **Replication** between accounts (copying/syncing data — useful for disaster recovery).
- Centralized account creation/management by an organization admin.

**Analogy:** Account = one branch office. Organization = the head office overseeing all branches.

**Interview angle:** *"Difference between an organization and an account?"*
An account is where you actually work with data (databases, warehouses, queries). An organization is the umbrella for centralized administration/billing across multiple accounts — you don't run queries "in" an organization.

---
**Next Topic:** Snowsight web interface
