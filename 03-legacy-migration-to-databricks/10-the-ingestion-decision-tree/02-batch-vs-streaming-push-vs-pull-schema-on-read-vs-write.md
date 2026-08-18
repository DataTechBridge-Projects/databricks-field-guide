---
title: "Batch vs Streaming, Push vs Pull, Schema-on-Read vs Write"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/batch-vs-streaming-push-vs-pull-schema-on-read-vs-write/
read_minutes: 3
---

# Batch vs Streaming, Push vs Pull, Schema-on-Read vs Write
{: .no_toc }

*Estimated read: 3 min*

The data ferry contract from the previous lecture -- schedule, capacity, manifest -- has to be
implemented as a real pipeline, and that means answering three independent design questions before
writing a single line of ingestion code. They're independent because a legacy migration tends to
conflate them: "batch" gets equated with "pull" and "streaming" with "push," when in practice all
three axes can mix and match.

## Batch vs. streaming

This is a question of **latency requirement**, not technical sophistication. A legacy nightly
extract-transform-load job that fed a morning report is a batch workload, full stop -- migrating it
to a 24/7 streaming pipeline adds operational complexity (checkpoint management, always-on compute)
that buys nothing if nobody reads the data before 8 AM anyway. On Databricks:

- **Batch** is a Lakeflow Jobs task on a schedule, reading a full or incremental snapshot and
  terminating when done.
- **Streaming** is [Structured Streaming](https://docs.databricks.com/aws/en/structured-streaming/)
  running continuously or on a short trigger interval, maintaining checkpoint state between runs.

Databricks blurs this line more than most legacy platforms did: a Structured Streaming job with
`Trigger.AvailableNow` behaves like a batch job (processes what's there, then stops) while reusing
the exactly-once, checkpoint-tracked machinery of streaming -- often the best default for
incremental loads that don't need continuous processing.

## Push vs. pull

- **Pull** means Databricks reaches out to the source on its own schedule -- a JDBC query against
  an Oracle table, a REST API poll. The legacy DBA is used to this model: it's the same shape as a
  BTEQ export job or a Talend pull connector, just triggered from Lakeflow Jobs instead of a cron
  entry.
- **Push** means the source system delivers data without being asked -- files landing in cloud
  storage from an upstream export process, a CDC tool streaming row-level changes into a queue.
  Auto Loader is built for exactly this model: it watches a cloud storage location and picks up new
  files as they arrive, with no polling query against the source system at all.

The practical decision driver is usually **who controls the source system**. If your migration team
owns the extract process, pull is simpler to reason about. If the source is a vendor SaaS system or
a database team that won't grant you direct query access, push (via an export process or a partner
CDC tool) is often the only option available.

## Schema-on-read vs. schema-on-write

Legacy warehouses enforced **schema-on-write**: a `CREATE TABLE` statement with fixed column types,
and any row that didn't conform was rejected at load time. Databricks supports both, and picking
correctly matters for ingestion specifically:

| | Schema-on-write | Schema-on-read |
|---|---|---|
| When schema is checked | At write time, against a defined table schema | At query time, inferred or applied on top of raw files |
| Legacy equivalent | Oracle `CREATE TABLE` + `INSERT` constraint checks | An external table over unstructured log files |
| Databricks mechanism | Delta Lake's schema enforcement on `MERGE`/`INSERT` | Reading raw JSON/CSV directly, or Auto Loader's schema inference in the bronze layer |
| Best fit | Silver and gold layers, where structure is expected and violations should fail loudly | Bronze/raw ingestion, where the source schema may drift and you want to capture data first, reconcile structure later |

{: .important }
> The medallion architecture resolves this trade-off rather than forcing a single choice: land data
> schema-on-read in bronze (accept what arrives, even if it's messier than expected) and enforce
> schema-on-write moving into silver, where Delta's schema enforcement guards against a malformed
> upstream change silently corrupting a table business users query directly.

Getting these three axes right for a given source table is the setup for the next lecture's harder
question -- once you know it's a pull, batch-friendly, schema-on-write-eventually workload, is JDBC
bulk extraction actually the right tool, or does the data's volume push you toward Auto Loader
instead?

<!-- prevnext:start -->

---

| [&larr; Previous: The Data Ferry: Schedule, Capacity, Manifest]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/the-data-ferry-schedule-capacity-manifest/' | relative_url }}) | [Next: JDBC Bulk vs Auto Loader: The 10TB Table Decision &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/jdbc-bulk-vs-auto-loader-the-10tb-table-decision/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

