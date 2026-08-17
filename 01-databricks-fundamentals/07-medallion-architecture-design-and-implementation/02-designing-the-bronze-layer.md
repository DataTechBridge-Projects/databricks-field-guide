---
title: "Designing the Bronze Layer"
parent: "Medallion Architecture - Design and Implementation"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-bronze-layer/
read_minutes: 9
---

# Designing the Bronze Layer
{: .no_toc }

*Estimated read: 9 min*

Bronze's entire job is to be a faithful, queryable copy of the source -- resist the urge to clean
anything here. This lecture covers what "minimal transformation" actually means in practice, and
the specific things bronze *should* do despite otherwise staying close to raw.

## What bronze should preserve

```sql
CREATE TABLE bronze.orders (
    raw_payload       VARIANT,
    _source_system    STRING,
    _ingested_at       TIMESTAMP,
    _source_file       STRING
)
USING DELTA;
```

- **The data, as close to source shape as practical.** If the source is JSON with a shifting
  schema, land it as `VARIANT` (Section 5) rather than forcing a rigid schema you'll have to keep
  adjusting.
- **Ingestion metadata**, prefixed distinctly (a common convention is a leading underscore):
  `_ingested_at`, `_source_file`, `_source_system`. This is what makes bronze auditable -- you can
  always answer "when did this row arrive, and from where," independent of anything the source
  system itself tracks.
- **Everything, including rows silver will later reject.** Bronze is not the place to filter out
  bad data -- a row with a null required field or an out-of-range value still lands in bronze,
  because bronze's job is fidelity to the source, not correctness.

## What bronze should *not* do

- **No deduplication.** Duplicates are a silver-layer concern -- removing them in bronze destroys
  your ability to audit "did the source actually send this twice."
- **No business logic.** No joins, no calculated fields beyond ingestion metadata, no filtering
  based on business rules.
- **No type coercion beyond what's structurally necessary.** If a source sends a string that looks
  like a number, bronze doesn't need to cast it -- that's silver's validation responsibility, and
  forcing a cast in bronze can silently swallow a data-quality signal (a malformed value that
  *should* fail validation later).
{: .important }

## Append-only, almost always

Bronze tables are overwhelmingly **append-only** -- new data lands as new rows, existing bronze
rows are essentially never updated or deleted (with the narrow exception of a deliberate
retention/compliance policy). This is what preserves bronze's role as an audit trail: if bronze
rows could be silently modified, "what did we actually receive" stops being answerable.

## Partitioning and file layout

For high-volume bronze tables, partitioning by ingestion date is a common, effective default:

```sql
CREATE TABLE bronze.orders (...)
USING DELTA
PARTITIONED BY (ingestion_date);
```

This keeps nightly incremental writes scoped to a small number of partitions, and makes "show me
everything that landed yesterday" a fast, partition-pruned query -- useful both for debugging and
for the incremental processing patterns Section 8 (Lakeflow Connect) builds on top of bronze.

## Coming from a legacy staging schema

If your prior warehouse had a staging schema you truncated and reloaded nightly, the biggest
mental shift here is that **bronze is not transient** -- it's a permanent, growing, queryable
history, not a scratch space cleared after each load. That permanence is exactly what makes time
travel (Section 5) and audit trails meaningful at the bronze layer specifically.

<!-- prevnext:start -->

---

| [&larr; Previous: What is Medallion Architecture]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/what-is-medallion-architecture/' | relative_url }}) | [Next: Designing the Silver Layer &rarr;]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-silver-layer/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
