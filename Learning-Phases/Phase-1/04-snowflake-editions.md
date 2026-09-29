# Phase 1 | Topic 4: Snowflake Editions

Snowflake is sold in tiers (editions), each unlocking more features — mainly around security, governance, and data protection — and costing more per compute credit as you go up.

## 1. Standard Edition
- Core features: virtual warehouses, storage, basic SQL, data sharing, VARIANT type, 1-day Time Travel.
- Can only **scale up** a single warehouse manually — no auto **scale out** (no multi-cluster).
- Good for smaller teams or learning, without heavy compliance needs.

## 2. Enterprise Edition (adds to Standard)
- **Extended Time Travel:** up to 90 days (vs 1 day in Standard) — lets you query/restore older data states.
- **Multi-cluster warehouses:** auto-adds clusters to handle concurrency ("scale out") — not available in Standard.
- **Materialized views** and other advanced performance features.

## 3. Business Critical Edition (adds to Enterprise)
- Higher-grade encryption and network security (e.g., enforced private connectivity, so traffic never crosses the public internet).
- **Database failover/failback (disaster recovery):** if an entire cloud region goes down, data can fail over to a backup region.
- Meets stricter compliance standards (HIPAA, PCI-DSS).
- Still shares the same underlying compute/storage infrastructure pool as other customers — just **logically** separated, not physically.

## 4. Virtual Private Snowflake (VPS) — highest tier
- Runs on **completely separate, dedicated infrastructure**, fully isolated from other customers (physically, not just logically).
- Used by the most security-sensitive organizations (large banks, government). Very expensive, rarely seen in typical DE jobs.

## Analogy
- Standard = shared apartment building
- Enterprise = nicer building, more shared amenities (multi-cluster, longer history)
- Business Critical = gated community with security guards and backup power (compliance, failover)
- VPS = your own private house, nobody else's utilities touch yours

## Interview Shortcut: Matching Scenario to Edition
| Scenario mentions... | Edition |
|---|---|
| Small team, learning, basic use case | Standard |
| Multi-cluster needed OR longer Time Travel for audit/compliance | Enterprise |
| Healthcare/finance/government, HIPAA/PCI-DSS, disaster recovery across regions | Business Critical |
| Extremely sensitive data, fully isolated infrastructure, cost not a concern | VPS |

**Also asked:** Can you upgrade editions later without migrating data? — Yes, it's a configuration change on the existing account, handled by Snowflake support; no data migration needed.

---
**Next Topic:** Supported cloud providers and regions
