---
title: "Check Your Knowledge"
parent: "Medallion Architecture - Design and Implementation"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- bronze, silver, gold layer design, and the
tradeoffs behind them.

1. What is bronze's primary responsibility?
   A. Aggregating data for dashboards
   B. Preserving a faithful, near-raw, auditable copy of the source
   C. Enforcing strict business rules
   D. Deduplicating records

2. Why should bronze avoid deduplication and business logic?
   A. It's too slow to do in bronze
   B. Doing so would destroy bronze's role as an accurate audit trail of what was actually received
   C. Deduplication is only possible in gold
   D. Bronze tables cannot run SQL

3. What is silver's primary responsibility?
   A. Storing raw files exactly as received
   B. Typing, validating, deduplicating, and conforming data to a stable, trustworthy schema
   C. Pre-aggregating data for a specific dashboard
   D. Managing cluster compute

4. What is the difference between the "report-only" and "quarantine" data quality strategies?
   A. They are the same thing
   B. Report-only flags issues while letting rows flow through; quarantine diverts bad rows to a separate table entirely
   C. Quarantine deletes bad rows permanently
   D. Report-only is only used in gold

5. Why is a gold table typically materialized rather than left as a view for a frequently-queried dashboard?
   A. Views are not supported in Delta Lake
   B. A materialized table avoids recomputing the same aggregation from scratch on every single query
   C. Materialized tables are always more accurate
   D. Views cannot join multiple tables

6. Why do multiple gold tables commonly derive from the same silver layer rather than each being built independently from bronze?
   A. Gold tables cannot read from bronze directly
   B. It avoids each consumer re-deriving cleaned data independently and risking inconsistent versions of the same facts
   C. Silver tables are faster to query than bronze
   D. It's a Unity Catalog technical requirement

7. What is the most common Medallion anti-pattern described in this section?
   A. Using too many gold tables
   B. Skipping a layer (e.g. building gold directly from bronze) under deadline pressure, creating compounding technical debt
   C. Partitioning bronze tables by date
   D. Using VARIANT columns in bronze

8. Where does SCD Type 2 dimension modeling typically live in the Medallion pattern?
   A. Bronze
   B. Silver
   C. Gold
   D. It doesn't fit into the Medallion pattern

9. What ingestion metadata is commonly added to bronze tables?
   A. Business rule flags
   B. Aggregated totals
   C. Fields like `_ingested_at` and `_source_file`, to track when and from where data arrived
   D. Customer tier classifications

10. What determines whether a gold output should be a table or a view?
    A. Table size alone
    B. Whether it's queried repeatedly with latency sensitivity (table) versus needing to always reflect live data for infrequent/exploratory use (view)
    C. Views are always better for gold
    D. Tables are only used in bronze

## Answer Key

1. **B** -- bronze exists to faithfully mirror the source with ingestion metadata, not to clean or aggregate.
2. **B** -- deduplicating or transforming in bronze would erase the audit trail of what was actually received.
3. **B** -- silver's job is typing, validation, deduplication, and conforming to a trustworthy schema.
4. **B** -- report-only flags and keeps rows; quarantine removes bad rows to a separate table.
5. **B** -- materializing avoids repeated, wasteful recomputation for frequently-viewed dashboards.
6. **B** -- a shared silver layer prevents each gold consumer from re-deriving and potentially diverging on the same facts.
7. **B** -- skipping a layer under pressure creates compounding debt, since downstream consumers inherit unvalidated data.
8. **B** -- SCD Type 2 history modeling is a silver-layer responsibility, typically via MERGE.
9. **C** -- ingestion metadata like `_ingested_at`/`_source_file` makes bronze auditable.
10. **B** -- the table-vs-view choice hinges on query frequency/latency needs versus the need for always-live data.

<!-- prevnext:start -->

---

| [&larr; Previous: Architecture Review and Design Decisions]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/architecture-review-and-design-decisions/' | relative_url }}) | [Next: Lakeflow Connect - Ingestion Pillar &rarr;]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

