---
title: "File-Based Ingestion with Auto Loader"
parent: "Lakeflow Connect - Ingestion Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/file-based-ingestion-with-auto-loader/
read_minutes: 15
---

# File-Based Ingestion with Auto Loader
{: .no_toc }

*Estimated read: 15 min*

Not every source is a SaaS API or a database -- flat files landing in cloud storage (CSVs from a
partner, JSON exports, Parquet drops from another system) are still one of the most common
ingestion patterns in any real data platform. **Auto Loader** is Databricks' purpose-built tool
for this: incremental file ingestion, without the connector infrastructure of the previous three
lectures.

## The `cloudFiles` source

```python
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")
      .option("cloudFiles.schemaLocation", "/Volumes/main/checkpoints/orders_schema/")
      .load("/Volumes/main/landing/orders/"))

(df.writeStream
   .format("delta")
   .option("checkpointLocation", "/Volumes/main/checkpoints/orders_stream/")
   .trigger(availableNow=True)
   .table("bronze.orders"))
```

`cloudFiles` is a Structured Streaming source (the streaming API from Section 5, applied
specifically to files) that detects new files as they arrive and processes only what hasn't been
seen before -- tracked via the checkpoint, exactly like any other streaming write.

## Two file detection modes

- **Directory listing** -- periodically lists the source directory to find new files. Simple, no
  extra cloud infrastructure, works well for low-to-moderate file volumes.
- **File notification** -- subscribes to cloud storage event notifications (S3 event
  notifications via SQS on AWS) so new files are detected the moment they land, without repeatedly
  listing the whole directory. Scales better for very high file-arrival volumes, at the cost of
  extra AWS infrastructure (an SQS queue) to configure.

```python
.option("cloudFiles.useNotifications", "true")  # switch to file notification mode
```

For most learning and moderate-production workloads, directory listing (the default) is
sufficient -- reach for notification mode specifically when directory listing itself becomes a
bottleneck at high file-arrival rates.

## Schema inference and evolution

```python
.option("cloudFiles.schemaLocation", "/Volumes/main/checkpoints/orders_schema/")
.option("cloudFiles.inferColumnTypes", "true")
.option("cloudFiles.schemaEvolutionMode", "addNewColumns")
```

Auto Loader can **infer** a schema from a sample of the source files (persisted at
`schemaLocation`, so it's not re-inferred on every run) and **evolve** that schema automatically as
new columns appear in later files -- directly relevant to the next lecture's production concerns
around schema drift.

## A realistic bronze ingestion pipeline

```python
raw_stream = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "csv")
    .option("cloudFiles.schemaLocation", "/Volumes/main/checkpoints/orders_schema/")
    .option("header", "true")
    .load("/Volumes/main/landing/orders/"))

bronze_stream = raw_stream.selectExpr(
    "*",
    "_metadata.file_path AS _source_file",
    "current_timestamp() AS _ingested_at"
)

(bronze_stream.writeStream
    .format("delta")
    .option("checkpointLocation", "/Volumes/main/checkpoints/orders_stream/")
    .trigger(availableNow=True)
    .table("bronze.orders"))
```

Note the `_metadata.file_path` and `_ingested_at` columns -- exactly the bronze ingestion metadata
pattern from Section 7's Medallion design, populated automatically as part of the Auto Loader
read.

## Why not just a plain batch `spark.read` over the directory?

| Plain batch read of a directory | Auto Loader (`cloudFiles`) |
|---|---|
| Reprocesses every file, every run, unless you hand-write filtering logic | Only processes new files automatically |
| No built-in exactly-once guarantee across reruns | Checkpoint-tracked, exactly-once |
| Manual schema handling | Built-in inference + evolution |
| Doesn't scale gracefully to very high file counts | Designed specifically for high-volume, high-frequency file arrival |

For the complete official reference, including performance tuning and cost-optimization guidance
beyond this lecture's scope, see
[What is Auto Loader?](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)

<!-- prevnext:start -->

---

| [&larr; Previous: Incremental CDC from Database Using Managed Connector]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/incremental-cdc-from-database-using-managed-connector/' | relative_url }}) | [Next: Incremental Ingestion, Schema Evolution and Bad Records in Production &rarr;]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/incremental-ingestion-schema-evolution-and-bad-records-in-production/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
