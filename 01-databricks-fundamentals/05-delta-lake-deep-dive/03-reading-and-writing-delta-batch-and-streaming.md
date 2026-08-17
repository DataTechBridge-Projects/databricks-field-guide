---
title: "Reading and writing Delta - batch and streaming"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/reading-and-writing-delta-batch-and-streaming/
read_minutes: 14
---

# Reading and writing Delta - batch and streaming
{: .no_toc }

*Estimated read: 14 min*

Every Delta table you'll touch in this guide can be read and written **either** as a static batch
snapshot **or** as a continuously-updating stream -- using nearly identical code either way. This
lecture covers both, and the practical question of when each fits.

## Batch reads and writes

```python
# Batch read: a snapshot as of right now
df = spark.read.format("delta").table("main.default.orders")

# Batch write
(df.write
   .format("delta")
   .mode("append")
   .saveAsTable("main.default.orders"))
```

This is what you already expect: read a table, get back exactly what's committed at the moment
you read it, regardless of what happens afterward. Every example in the previous two lectures was
this batch model.

## Streaming reads: a table as a continuous feed

```python
stream_df = (spark.readStream
             .format("delta")
             .table("main.default.orders"))
```

The only syntactic difference from a batch read is `readStream` instead of `read` -- but the
semantics change completely. Instead of a fixed snapshot, `stream_df` represents **new rows as
they're committed** to the source table, processed incrementally rather than reprocessing
everything on each run.

```python
(stream_df.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/Volumes/main/checkpoints/orders_silver/")
    .trigger(availableNow=True)
    .table("main.default.orders_silver"))
```

**Key term:** the **checkpoint location** is where Spark Structured Streaming tracks exactly which
input has already been processed -- restart the stream and it resumes from there, rather than
reprocessing from the beginning or losing track entirely. This is the mechanism that makes
streaming **exactly-once** in practice, and it's specific to *this* stream/sink pairing -- reusing
a checkpoint location across unrelated streams will produce wrong results.
{: .important }

## Trigger modes: how "streaming" actually runs

Structured Streaming doesn't require an always-on process -- `trigger` controls the cadence:

| Trigger | Behavior |
|---|---|
| `trigger(availableNow=True)` | Process everything currently available, then stop -- effectively "incremental batch" |
| `trigger(processingTime="5 minutes")` | Wake up every 5 minutes, process what's new, repeat |
| Continuous (no trigger specified) | Process new data as it arrives, near-continuously |

For most data engineering pipelines in this guide -- including everything in Part 2's StepRight
project -- `availableNow=True` on a scheduled job (via Lakeflow Jobs, Section 10) is the practical
default: you get streaming's incremental processing semantics without paying for an always-on
cluster.

## Why use the streaming API for batch-shaped work at all?

This is the part worth sitting with if you're coming from a warehouse world where batch and
streaming were entirely separate tools. The streaming API's real value here isn't "near real-time"
-- it's **automatic incremental processing**: Spark tracks what's already been read via the
checkpoint, so each run only processes genuinely new data, without you hand-writing watermark or
high-water-mark logic the way you likely did in a legacy incremental-load ETL job.

```text
Legacy incremental load:
  1. Query max(updated_at) already processed
  2. WHERE updated_at > last_max_watermark
  3. Manually track and persist the new watermark

Delta streaming read + checkpoint:
  1. readStream, process, write
  2. Checkpoint tracks progress automatically
  3. Rerun -- only new data processes, no manual watermark logic
```

## Batch vs. streaming: which for which

| Use batch when | Use streaming (with `availableNow`) when |
|---|---|
| One-off analysis or backfill | Regular incremental pipeline runs |
| Full-table transformations that need the whole dataset | You want automatic new-data-only processing |
| Small, infrequently-updated reference/dimension tables | High-volume fact tables updated continuously |

You'll see this exact choice made explicitly in Section 8 (Lakeflow Connect) and Section 9
(Lakeflow Declarative Pipelines), where **streaming tables** are the primary building block for
bronze and silver layers specifically because of this incremental-by-default behavior.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating and Managing Delta tables]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/creating-and-managing-delta-tables/' | relative_url }}) | [Next: Time Travel - querying historical versions &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/time-travel-querying-historical-versions/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
