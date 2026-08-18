---
title: "JDBC Bulk vs Auto Loader: The 10TB Table Decision"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/jdbc-bulk-vs-auto-loader-the-10tb-table-decision/
read_minutes: 2
---

# JDBC Bulk vs Auto Loader: The 10TB Table Decision
{: .no_toc }

*Estimated read: 2 min*

Here's a scenario every migration architect eventually hits: a 10TB Oracle fact table needs to land
in the bronze layer, and the obvious first instinct -- "just JDBC it over" -- turns out to be the
wrong default at that size. Working through why clarifies when each tool actually earns its place.

## JDBC bulk: pulling directly from the source

[Databricks' JDBC support](https://docs.databricks.com/aws/en/connect/external-systems/jdbc)
reads a source table directly into a DataFrame, with parallelism controlled by `numPartitions`,
`partitionColumn`, `lowerBound`, and `upperBound`:

```python
df = (spark.read
      .format("jdbc")
      .option("url", jdbc_url)
      .option("dbtable", "orders")
      .option("numPartitions", 16)
      .option("partitionColumn", "order_id")
      .option("lowerBound", 1)
      .option("upperBound", 50_000_000)
      .load())
```

This is the direct migration path for a legacy BTEQ or SQL*Plus export job, and it works well for
tables in the gigabytes-to-low-terabytes range with a numeric column suitable for partitioning. At
10TB, though, two constraints bite hard:

- **Source-side load.** Sixteen (or more) parallel JDBC connections each running a range-scan query
  against the same production Oracle table compete directly with OLTP traffic on that database --
  exactly the "capacity" term of the ferry contract from the first lecture. A DBA who agreed to a
  nightly export window did not agree to a query that saturates I/O on the live system.
  - **No incremental story.** A JDBC pull re-reads the full table (or requires you to hand-roll a
  watermark column and `WHERE` clause) every run; there's no built-in notion of "only the rows that
  changed since last time" the way file-based ingestion has.

## Auto Loader: incremental file ingestion

[Auto Loader](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)
solves a different problem: it watches a cloud storage location and incrementally, exactly-once
processes new files as they land, using the `cloudFiles` Structured Streaming source:

```python
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "parquet")
      .option("cloudFiles.schemaLocation", "/Volumes/bronze/orders/_schema")
      .load("/Volumes/bronze/orders/incoming/"))
```

For the 10TB table, this reframes the problem: instead of one enormous pull, the source system
exports the table once (or incrementally) as files into cloud storage, and Auto Loader picks up
each file exactly once, tracked via RocksDB-backed checkpoint state -- no re-scanning the whole
table on every run, and no sustained load against the production source database.

## The decision, side by side

| | JDBC bulk | Auto Loader |
|---|---|---|
| Reads from | Live source database directly | Files already landed in cloud storage |
| Source-system load | Sustained, scales with parallelism | None -- decoupled after export |
| Incremental support | Manual (watermark column + `WHERE`) | Built-in, exactly-once via checkpoints |
| Best fit | Small-to-mid tables, one-time or infrequent full loads | Large tables, ongoing ingestion, files delivered continuously |
| Legacy equivalent | A BTEQ/SQL*Plus export query | A file-drop landing zone with a watcher process |

{: .important }
> JDBC bulk and Auto Loader aren't mutually exclusive on the same migration -- a common pattern
> uses one JDBC pull for the initial historical backfill of a 10TB table, then switches to an
> export-to-cloud-storage-plus-Auto-Loader pattern (or a CDC tool, covered next) for ongoing
> incremental loads once the backfill is done.

That handoff -- from a one-time bulk load to an ongoing low-latency feed -- is exactly the gap the
partner CDC tools in the next lecture are built to fill when neither JDBC polling nor a manual file
export is fast enough.

<!-- prevnext:start -->

---

| [&larr; Previous: Batch vs Streaming, Push vs Pull, Schema-on-Read vs Write]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/batch-vs-streaming-push-vs-pull-schema-on-read-vs-write/' | relative_url }}) | [Next: Partner Tools: Fivetran, Qlik, Arcion for CDC &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/partner-tools-fivetran-qlik-arcion-for-cdc/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

