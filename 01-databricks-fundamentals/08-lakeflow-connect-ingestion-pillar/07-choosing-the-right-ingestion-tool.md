---
title: "Choosing the Right Ingestion Tool"
parent: "Lakeflow Connect - Ingestion Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/choosing-the-right-ingestion-tool/
read_minutes: 4
---

# Choosing the Right Ingestion Tool
{: .no_toc }

*Estimated read: 4 min*

A short, direct decision guide, pulling together every pattern from this section into one table --
worth bookmarking for the next time you're scoping a new ingestion source.

## The decision table

| Source type | First choice | Reach for instead when |
|---|---|---|
| Supported SaaS app (Salesforce, HubSpot, Jira, Workday, GitHub, ...) | Managed SaaS connector | The specific app isn't supported -- then a custom API integration |
| Relational database, deletes matter, CDC access available | CDC-based managed connector | Source team can't grant replication access -- then query-based |
| Relational database, deletes don't matter or CDC access unavailable | Query-based connector | You genuinely need delete visibility -- go back to CDC, or add periodic reconciliation |
| Files landing in cloud storage | Auto Loader (`cloudFiles`) | Extremely low, one-off volume -- a plain batch read may be simpler for a true one-time load |
| Message stream (Kafka, Kinesis) | Streaming connector | Not covered in this guide's Part 1 -- same Structured Streaming principles apply |

## Three questions that settle most decisions

1. **Does the source have a managed connector?** If yes, start there -- custom-built ingestion is
   worth its added maintenance burden specifically when no managed option exists, not as a default.
2. **Do you need to see deletes?** This is the single biggest factor distinguishing CDC from
   query-based database ingestion -- decide this explicitly, don't default to whichever is easier
   to set up without checking whether delete visibility actually matters for the use case.
3. **What access can the source team actually grant you?** CDC's replication-level access request
   is a real conversation with a real DBA or platform team -- know before you design the pipeline
   whether that access is realistically obtainable, rather than designing for CDC and discovering
   the access request gets denied.

## What's next

This closes out Lakeflow Connect. Data now lands reliably in bronze -- Section 9, **Lakeflow
Declarative Pipelines**, is where it gets transformed into silver and gold, replacing the manual
`MERGE`/streaming code you've seen in isolated examples so far with a declarative framework
purpose-built for exactly this.

<!-- prevnext:start -->

---

| [&larr; Previous: Incremental Ingestion, Schema Evolution and Bad Records in Production]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/incremental-ingestion-schema-evolution-and-bad-records-in-production/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
