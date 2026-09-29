# Phase 1 | Topic 5: Supported Cloud Providers and Regions

## Cloud Providers
Snowflake doesn't own physical data centers — it runs on top of one of three major cloud providers, chosen at account creation:
1. **AWS** (Amazon Web Services)
2. **Microsoft Azure**
3. **Google Cloud Platform (GCP)**

**Why the choice matters**
- Determines which **regions** are available.
- Affects integration smoothness with other tools already used (e.g., Azure Data Factory/Blob Storage pairs naturally with a Snowflake-on-Azure account).
- Slight pricing differences, since Snowflake credit cost is influenced by the underlying provider's own pricing.

**Key rule:** once chosen, the cloud provider **cannot be switched** for an existing account — because data is physically stored on that provider's infrastructure. Moving providers means creating a new account on the target cloud and migrating the data across (often via Snowflake's replication/sharing features) — not a configuration change.

## Regions
A **region** = a specific physical geographic location of a cloud provider's data centers (e.g., Azure "Central India," "East US").

**Why regions matter**
- **Latency:** a region closer to users/company means faster queries (shorter physical distance).
- **Data residency/compliance:** some countries require certain data to stay stored within their borders (e.g., data protection laws, GDPR in Europe).
- **Cost:** pricing can vary slightly by region.

Region availability depends on the chosen cloud provider — the supported region list differs across AWS, Azure, and GCP (check Snowflake's docs for the current list).

**Availability Zones:** within a region, providers have multiple separate physical data center buildings. Snowflake automatically spreads data across these for reliability — no manual management needed.

## Interview Angle
*"Why might a company running its app in Azure's 'Central India' region also choose that region for Snowflake?"*
To reduce latency (app and Snowflake data don't travel far) and to stay compliant with data residency laws for Indian customer data.

*"Can you migrate a Snowflake account from one cloud provider to another?"*
Not directly — requires a new account on the target cloud and migrating the data.

---
**Next Topic:** Organizations and accounts
