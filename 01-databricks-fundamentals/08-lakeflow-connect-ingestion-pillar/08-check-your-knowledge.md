---
title: "Check Your Knowledge"
parent: "Lakeflow Connect - Ingestion Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- Lakeflow Connect's connector types, Auto Loader,
schema drift, and bad records.

1. What happens on the first run of a Lakeflow Connect ingestion pipeline, versus subsequent runs?
   A. Every run reprocesses all data from scratch
   B. The first run ingests all available data; subsequent runs ingest only what's new
   C. Only the first run works; subsequent runs must be manually reconfigured
   D. Subsequent runs always overwrite the destination table

2. Why can query-based (cursor-column) connectors struggle to capture deletes?
   A. They don't support SQL
   B. A physically deleted source row leaves no trace for a cursor query to find
   C. Deletes are always captured automatically
   D. Cursor columns cannot be timestamps

3. What does a CDC-based connector read from the source database?
   A. Only the current table snapshot
   B. The database's own change stream (e.g. binlog, logical replication)
   C. A manually maintained changelog table
   D. Nothing -- it polls like a query-based connector

4. What is the `cloudFiles` Structured Streaming source used for?
   A. Reading from Kafka
   B. Incremental, checkpoint-tracked ingestion of files landing in cloud storage (Auto Loader)
   C. Connecting to SaaS APIs
   D. Running SQL warehouses

5. What is the difference between directory listing and file notification detection modes in Auto Loader?
   A. They produce different data
   B. Directory listing periodically lists the source directory; file notification uses cloud storage events for near-immediate detection at higher scale
   C. File notification is always slower
   D. Directory listing requires an SQS queue

6. What does the `rescue` schema evolution mode do?
   A. Fails the entire pipeline on any schema change
   B. Captures unexpected/differently-shaped data into a `_rescued_data` column while the pipeline keeps running
   C. Silently drops new columns
   D. Automatically deletes malformed records

7. What is a "bad record" as distinct from a schema drift issue?
   A. A record that is simply malformed and cannot be parsed at all, routed to `badRecordsPath`
   B. Any record with a new column
   C. A duplicate row
   D. A record missing ingestion metadata

8. Why does a database user configured for CDC typically need more privileges than one used for query-based ingestion?
   A. CDC requires elevated replication-level access to read the change stream, not just standard query access
   B. CDC connectors require admin rights to the entire database server
   C. There is no difference in required privileges
   D. Query-based connectors require more privileges

9. According to this section's decision guide, what should you check first when scoping a new ingestion source?
   A. Whether a managed connector already supports it
   B. The size of the source table
   C. Which programming language the source uses
   D. Whether the source has a REST API

10. What ingestion metadata columns are typically added automatically to a bronze table sourced via Lakeflow Connect or Auto Loader?
    A. Business rule flags
    B. Fields like `_ingested_at` and source-tracking columns (e.g. `_source_file`, `_pipeline_run_id`)
    C. Aggregated summary statistics
    D. Customer tier classifications

## Answer Key

1. **B** -- the first run ingests everything available; every later run is incremental.
2. **B** -- a deleted row simply disappears from the source, leaving nothing for a cursor query to detect.
3. **B** -- CDC connectors read the database's own change stream directly.
4. **B** -- `cloudFiles` is Auto Loader's Structured Streaming source for incremental file ingestion.
5. **B** -- directory listing polls the directory; file notification uses cloud events for faster detection at scale.
6. **B** -- rescue mode captures unexpected data into `_rescued_data` without stopping the pipeline.
7. **A** -- a bad record fails to parse entirely and is routed to `badRecordsPath`, distinct from a differently-shaped-but-parseable schema drift case.
8. **A** -- CDC needs replication-level access to read the change stream, a higher privilege tier than standard queries.
9. **A** -- checking for an existing managed connector first avoids unnecessary custom-built ingestion.
10. **B** -- ingestion metadata like `_ingested_at` and source-tracking fields are added automatically, following bronze design principles.

<!-- prevnext:start -->

---

| [&larr; Previous: Choosing the Right Ingestion Tool]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/choosing-the-right-ingestion-tool/' | relative_url }}) | [Next: Lakeflow Spark Declarative Pipelines - Transformation Pillar &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

