# Topic 4 (Hands-On Practice) + External Staging Walkthrough

## Part 1: Snowflake Trial Account Setup

- Signed up at signup.snowflake.com
- Cloud provider chosen: **Azure** (not AWS — noted for later, since External Stage syntax will use Azure terms, not S3)
- Edition: **Standard** (not Enterprise) — key difference: Time Travel is limited to **1 day** instead of up to 90 days. Not an issue for practice/learning.
- Logged into Snowsight (Snowflake's web UI) — SQL is written and run inside **Workspace** (also called Worksheets in some versions).
- Confirmed default warehouse `COMPUTE_WH` is active by running:
```sql
SELECT CURRENT_WAREHOUSE(), CURRENT_ACCOUNT(), CURRENT_VERSION();
```

---

## Part 2: Building the Database Structure (from Topic 3, hands-on)

```sql
CREATE DATABASE SALES_DB;
CREATE SCHEMA SALES_DB.RAW;
CREATE SCHEMA SALES_DB.STAGING;
CREATE SCHEMA SALES_DB.ANALYTICS;
```
Every database also auto-creates two schemas without you doing anything: `PUBLIC` and `INFORMATION_SCHEMA`.

Created the RAW landing table (all columns as STRING, matching the Bronze/Medallion-architecture idea — RAW = Bronze layer, preserving original data exactly as received, since forcing strict types immediately risks load failures on messy source data):
```sql
CREATE TABLE SALES_DB.RAW.ORDERS_RAW (
    order_id STRING,
    customer_id STRING,
    order_date STRING,
    amount STRING
);
```

Manually inserted first test rows (before practicing file-based loading):
```sql
INSERT INTO SALES_DB.RAW.ORDERS_RAW (order_id, customer_id, order_date, amount)
VALUES 
('1001', 'C001', '2026-09-01', '250.00'),
('1002', 'C002', '2026-09-02', '175.50'),
('1003', 'C001', '2026-09-03', '99.99');
```

---

## Part 3: Practicing Stages

Created a Named Internal Stage:
```sql
CREATE STAGE my_internal_stage;
```

**Key learning — visibility of stage types:**
- Named Stages appear in the Snowsight sidebar under "Stages."
- **User Stages and Table Stages do NOT appear in the sidebar** — they exist automatically but are accessed only via special reference syntax:
  - Table Stage: `@%TABLE_NAME` (e.g., `@%ORDERS_RAW`)
  - User Stage: `@~`
  - Named Stage: `@my_internal_stage`

Verified a table stage exists (even though invisible in UI) by listing it — empty result is a valid, correct outcome, not an error:
```sql
LIST @SALES_DB.RAW.%ORDERS_RAW;
```

**Uploading a file without SnowSQL CLI:** Since practice was done entirely in Snowsight (web UI), file upload into an internal stage was done via the UI directly (Stage → "+ Files" / Upload button) instead of running the `PUT` command from a command line.

Verified upload:
```sql
LIST @my_internal_stage;
```

---

## Part 4: File Formats and COPY INTO — Hands-On

Created a strict file format:
```sql
CREATE FILE FORMAT csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1;
```

Loaded a clean file (`orders_batch2.csv`) successfully:
```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_internal_stage
FILE_FORMAT = (FORMAT_NAME = csv_format)
ON_ERROR = 'CONTINUE';
```

**Key learning — Validation vs Cleaning are different concerns:**
- **Cleaning** (done later, in STAGING) = fixing business-logic/meaning problems — e.g., converting a text date into a real DATE type, standardizing formats.
- **Validation** (`VALIDATION_MODE`) = checking structural/parsing problems *before* data enters RAW — e.g., wrong column count, unescaped delimiters breaking the row shape. Even RAW (all-STRING columns) can't survive a structurally broken row — a wrong column count is unparseable regardless of data type.

---

## Part 5: Deliberately Breaking a File to Test Validation

Created and uploaded `orders_broken.csv` containing:
- A row with a **missing column** (too few values)
- A row with an **extra column** (too many values)

Ran validation on just this file:
```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_internal_stage
FILES = ('orders_broken.csv')
FILE_FORMAT = (FORMAT_NAME = csv_format)
VALIDATION_MODE = 'RETURN_ERRORS';
```
**Result:** both the missing-column row AND the extra-column row were flagged as errors — Snowflake's default (`ERROR_ON_COLUMN_COUNT_MISMATCH = TRUE`, the implicit default) requires an *exact* column count match; both too few and too many count as structural mismatches.

### Testing the Lenient Setting
```sql
CREATE OR REPLACE FILE FORMAT csv_format_lenient
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;
```
**Debugging lesson learned:** A semicolon (`;`) ends a SQL statement. Placing one mid-way through a multi-line `CREATE FILE FORMAT` command (e.g., right after `SKIP_HEADER = 1;`) silently cuts off the remaining options — they never become part of the object. Only the very last line of a statement should end with `;`.

Loaded the broken file using the lenient format:
```sql
COPY INTO SALES_DB.RAW.ORDERS_RAW
FROM @my_internal_stage
FILES = ('orders_broken.csv')
FILE_FORMAT = (FORMAT_NAME = csv_format_lenient)
ON_ERROR = 'CONTINUE'
FORCE = TRUE;
```
`FORCE = TRUE` was needed to override Snowflake's load history tracking, which would otherwise skip a file it believed was already processed (from the earlier strict-format attempt) — used here only for repeated practice/testing, not something to use in real pipelines.

**Result — confirmed hands-on:**
- Row with **missing column** → loaded successfully, missing field became **NULL**.
- Row with **extra column** → loaded successfully, the extra value was **silently dropped/ignored**.

**Real-world risk this exposes:** if that dropped extra field had contained meaningful business data (e.g., a discount code), it would be lost with zero warning. This is exactly why the strict default exists — lenient mode trades safety for flexibility, and should only be used when the data source and its risks are well understood.

**Cleanup after testing:**
```sql
DELETE FROM SALES_DB.RAW.ORDERS_RAW
WHERE order_id IN ('3001', '3002', '3003');
```

---

## Part 6: External Staging with Azure Blob Storage (Conceptual Walkthrough — not yet hands-on)

Hands-on practice was limited to Internal Stages, since External Staging requires an active Azure Storage Account (not yet set up). Below is the full conceptual flow to follow once Azure access is available.

### Concept Mapping: Generic → Azure Terms
| Generic Concept | Azure's Term |
|---|---|
| Cloud storage account | Azure **Storage Account** |
| A "bucket" (S3 equivalent) | Azure **Blob Container** |
| Secure trust mechanism | **Storage Integration** with Azure AD (tenant-based trust), or a SAS Token |

### Step-by-Step Flow
**1. Create a Storage Account + Blob Container in Azure Portal** — this is where files physically sit (e.g., `raw-data-container`).

**2. Create a Storage Integration in Snowflake:**
```sql
CREATE STORAGE INTEGRATION azure_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = '<your-azure-tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://mystorageaccount.blob.core.windows.net/raw-data-container/');
```
`AZURE_TENANT_ID` identifies which Azure Active Directory organization Snowflake should trust — Azure's equivalent of AWS's `STORAGE_AWS_ROLE_ARN`.

**3. Authorize Snowflake's identity inside Azure Portal** — Snowflake generates a service principal/consent request after step 2, which must be approved in Azure Portal to complete the two-way trust handshake.

**4. Create the External Stage:**
```sql
CREATE STAGE azure_external_stage
  URL = 'azure://mystorageaccount.blob.core.windows.net/raw-data-container/'
  STORAGE_INTEGRATION = azure_int;
```

**5. From here, everything is identical to Internal Stage practice already completed:**
- `LIST @azure_external_stage;`
- `COPY INTO ... FROM @azure_external_stage ...`
- No `PUT` step needed — files are uploaded to the Azure container separately (via Azure Portal, Azure Storage Explorer, or another system), not through Snowflake.

**Key takeaway:** Stages, File Formats, `COPY INTO`, `ON_ERROR`, `VALIDATION_MODE` all behave identically regardless of cloud provider. The only new piece with External Staging is the one-time secure handshake setup (Storage Integration + Azure-side authorization).

---

## Glossary — New Terms From This Session
| Term | Meaning |
|---|---|
| FORCE = TRUE | COPY INTO option that overrides load history tracking, reprocessing a file even if already loaded (testing/practice use only) |
| ERROR_ON_COLUMN_COUNT_MISMATCH | File format setting controlling whether mismatched column counts cause an error (TRUE, default) or are tolerated (FALSE) |
| SAS Token | Azure's Shared Access Signature — a time-limited credential for accessing blob storage |
| Azure Tenant ID | Identifier for an Azure Active Directory organization, used to establish trust with Snowflake |
| Service Principal | An Azure identity object representing an external application (like Snowflake) that has been granted access |

---
**Next Topic:** Semi-Structured Data — VARIANT type, JSON/Parquet handling
