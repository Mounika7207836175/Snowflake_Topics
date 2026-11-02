# Snowflake Notes: Topic 3, Architecture and Setup

## 1. Storage layer

- All table data sits in the cloud provider's object storage (Azure Blob/ADLS, S3, or GCS). You never touch the files directly. You reach them only through SQL.
- Data is cut into **micro-partitions**: small files holding roughly 50 to 500 MB of data (before compression).
- Inside each micro-partition, data is stored **column by column** (columnar) and **compressed**. A query needing 3 columns out of 100 reads only those 3.
- Micro-partitions are **immutable** (never edited in place). An UPDATE writes a new micro-partition and updates the metadata.
- Snowflake keeps metadata per micro-partition (min/max value per column, row count), which makes **pruning** possible.
- You pay for storage per TB per month, on the *compressed* size.

*Example:* a 1 TB raw table may compress to about 250 GB, so you pay for 250 GB.

## 2. Compute layer and virtual warehouses

A **virtual warehouse** is a cluster of servers that runs your queries. It stores no permanent data.

| Size | Servers | Credits/hour |
|---|---|---|
| X-Small | 1 | 1 |
| Small | 2 | 2 |
| Medium | 4 | 4 |
| Large | 8 | 8 |
| X-Large | 16 | 16 |
| 2X-Large | 32 | 32 |
| 3X-Large | 64 | 64 |
| 4X-Large | 128 | 128 |

Each step up doubles servers and cost. A perfectly parallel query runs about twice as fast.

**How a query runs**
1. The cloud services layer decides which micro-partitions are needed.
2. Those micro-partitions are divided among the servers. With 4 servers and 120 micro-partitions, each reads about 30. (Work is split by micro-partitions, not rows.)
3. Each server reads, filters, and partly aggregates its share.
4. Results are combined and returned.

**Local cache:** each server's local disk holds recently read micro-partitions, so repeated queries are faster. It disappears when the warehouse suspends. The disk is a cache, not where the data lives.

**Important settings**
- **AUTO_SUSPEND:** stops the warehouse after idle seconds (billing stops).
- **AUTO_RESUME:** wakes it when a query arrives.
- **Scale up** (bigger size): for one heavy query.
- **Scale out** (multi-cluster, Enterprise edition and above): for many users at once.
- Billing: per second, with a **60-second minimum** each time it starts.

*Scenario:* finance's month-end report takes 40 minutes on Medium. Resize to Large before running: about 20 minutes, with about the same total credits (2x speed at 2x price per hour). Then resize back down.

## 3. Cloud services layer

The "brain" of Snowflake. It runs on Snowflake-managed compute, not on your warehouses. It handles:
- **Authentication and access control** (logins, roles, privileges)
- **Metadata management** (micro-partition info, table structure)
- **Query parsing and optimization** (builds the plan, decides which micro-partitions to read)
- **Transaction management** (keeps data consistent)
- **Infrastructure management**

Billing note: cloud services usage is free up to 10% of your daily compute credits. You pay only above that.

## 4. Editions

| | Standard | Enterprise | Business Critical | VPS (Virtual Private Snowflake) |
|---|---|---|---|---|
| For | Basic use, learning, small teams | Large companies | Sensitive data (finance, health) | Highest-security organizations |
| Time Travel | Up to 1 day | Up to **90 days** | Up to 90 days | Up to 90 days |
| Multi-cluster warehouses | No | **Yes** | Yes | Yes |
| Extra features | Core features | Materialized views, column masking, and more | **Enhanced security**: customer-managed encryption keys, support for compliance standards (HIPAA, PCI), private connectivity | Everything above |
| Infrastructure | Shared with other customers | Shared | Shared | **Completely separate** Snowflake environment |

Each edition includes everything from the one before it. Higher editions cost more per credit.

*Scenarios:* a hospital storing patient records picks **Business Critical**. A startup building dashboards picks **Standard**. A bank that cannot share any infrastructure picks **VPS**.

## 5. Accounts and organizations

- **Account:** an isolated Snowflake environment with its own users, roles, databases, warehouses, and URL. It lives on **one cloud and one region**.
- **Organization:** a group of accounts owned by one company. The **ORGADMIN** role manages it, creates accounts, and sees billing and usage across all of them.

**Why use multiple accounts**
- Separate dev, test, and prod environments.
- Separate business units with their own billing and security.
- Place accounts on different clouds or regions (for example, an Azure account in Europe and an AWS account in the US).

## 6. Regions and clouds

When you create an account, you choose **a cloud** (AWS, Azure, or GCP) and **a region** (for example, Azure East US, or Central India).

Reasons to pick a region carefully:
- **Latency:** closer to your users and applications.
- **Data residency and compliance:** some laws (like GDPR in Europe) require data to stay in certain regions.
- **Near your source data:** moving data between clouds and regions can cost money (transfer fees) and time.
- **Cost and features:** prices and some features vary by region.

An account cannot be moved to another region later. To move, you replicate data to a new account.

## 7. Snowsight, SnowSQL, worksheets

**Snowsight** is Snowflake's web interface. Use it to run SQL, create warehouses, manage users, view query history and costs, and build dashboards.

**SnowSQL** is Snowflake's **command-line client**. Install it on your computer and type commands in a terminal.
- Used for: automation and scripts, scheduled jobs, running `.sql` files, and **loading local files** (the `PUT` command uploads a file from your computer to Snowflake storage).
- Snowflake also has a newer command-line tool called the Snowflake CLI, but SnowSQL is still common in interviews and older projects.
- It is **SnowSQL**, not NoSQL. NoSQL means non-relational databases like MongoDB.

**Worksheets** are the editor tabs in Snowsight where you write and run SQL. Each worksheet remembers its own **context**: role, warehouse, database, and schema. Snowflake may show a newer version called "Workspaces" in your account, built on the same idea.

*Before running any query, check the context* (role, warehouse, database, schema). A query fails with "no active warehouse" if you forget to select one.

---

## 8. Installing SnowSQL (do later on a laptop)

Download from Snowflake's official documentation (search "SnowSQL install Snowflake docs"). Versions change, so avoid third-party sites.

**Windows**
1. Download the `.msi` installer from the official page.
2. Run it with the default options.
3. Open Command Prompt or PowerShell and run `snowsql -v`. A version number means it worked.

**Mac**
1. Download the `.pkg` installer, or run `brew install --cask snowflake-snowsql` if you use Homebrew.
2. Open Terminal and run `snowsql -v`.

**Linux**
1. Download the `.bash` installer from the official page.
2. Run it in the terminal and follow the prompts.
3. Check with `snowsql -v`.

### Connecting
You need:
1. **Account identifier:** in Snowsight, open your account menu at the bottom left and copy the account identifier (usually `orgname-accountname`).
2. **Username**
3. **Password** (and MFA code, if your account asks for one)

```
snowsql -a <account_identifier> -u <username>
```

It asks for your password. When the prompt changes to `username#WAREHOUSE@DATABASE.SCHEMA>`, you are connected.

### Saving a connection (works on Windows too)
SnowSQL reads a config file:
- **Windows:** `C:\Users\<your-username>\.snowsql\config`
- **Mac/Linux:** `~/.snowsql/config`

If the file does not exist, create it. Add:

```
[connections.myconn]
accountname = <account_identifier>
username = <your_username>
warehouse = learn_wh
```

Then connect with:
```
snowsql -c myconn
```

It still asks for your password each time, which is safer. Do not store your password in plain text in this file.

### First commands to try
```sql
USE WAREHOUSE learn_wh;
SELECT CURRENT_ACCOUNT(), CURRENT_REGION(), CURRENT_VERSION();
```
- `!help` shows help.
- `!quit` or `!exit` leaves SnowSQL.

### Why SnowSQL exists (interview points)
- **Automation:** run `.sql` files or scheduled jobs from scripts, with no browser.
- **Loading local files:** `PUT` uploads a file from your laptop to Snowflake storage.
- **Repeatable work:** the same commands run the same way every time.

---

## Quick recap
1. Storage: micro-partitions in cloud storage, columnar, compressed, immutable.
2. Compute: virtual warehouses are clusters of servers. Size decides server count. No permanent data.
3. Cloud services: metadata, security, optimization, transactions.
4. Editions: Standard, Enterprise (90-day Time Travel, multi-cluster), Business Critical (security and compliance), VPS (dedicated environment).
5. An organization contains accounts. An account lives on one cloud and one region.
6. Snowsight = web UI, SnowSQL = command line, worksheet = SQL editor tab with its own context.

---

## Hands-on (do later on a laptop)

**1. Draw the query journey:** user writes a query, then cloud services (checks login, plans, picks micro-partitions), then warehouse servers (read in parallel), then results. Mark which layer does what.

**2. In Snowsight:** open a worksheet and run:
```sql
CREATE WAREHOUSE learn_wh
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

USE WAREHOUSE learn_wh;

SELECT CURRENT_ACCOUNT(), CURRENT_REGION(), CURRENT_VERSION();
```
Note your account's cloud, region, and edition.

**3. SnowSQL checklist**
1. Install and run `snowsql -v`.
2. Connect with `snowsql -a ... -u ...`, or save a connection and use `snowsql -c myconn`.
3. Run the SELECT above and write down your cloud and region.
4. Type `!quit`.
