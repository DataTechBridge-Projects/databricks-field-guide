---
title: "OPTIMIZE, VACUUM and Data Retention"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/optimize-vacuum-and-data-retention/
read_minutes: 16
---

# OPTIMIZE, VACUUM and Data Retention
{: .no_toc }

*Estimated read: 16 min*

Every `MERGE`, `UPDATE`, streaming append, and `DELETE` against a Delta table leaves files behind
-- superseded versions kept around for time travel, and, over time, many small files instead of a
few well-sized ones. `OPTIMIZE` and `VACUUM` are the two maintenance operations that keep a
frequently-written table healthy and cost-efficient. Neither is optional for a real production
table; both need to be understood before you run them, not after something goes wrong.

## The small-file problem

Streaming appends and incremental `MERGE` operations tend to write many small files over time
rather than a few large ones -- each micro-batch or merge run adds its own files. Query
performance degrades as file count grows, because Spark has more file-open overhead and less
efficient I/O per byte read, even though total data volume hasn't changed.

## `OPTIMIZE`: compaction

```sql
OPTIMIZE main.default.orders;

-- Optionally restrict to a subset, e.g. only recently-written partitions
OPTIMIZE main.default.orders WHERE order_date >= '2026-08-01';
```

`OPTIMIZE` rewrites small files into fewer, well-sized ones -- a **compaction** operation that
doesn't change the table's logical content at all, only its physical file layout. Combined with
**Z-Ordering** or **Liquid Clustering** (covered in depth as part of Part 3's physical-design
content, since it matters most at migration scale), `OPTIMIZE` also co-locates related data
together on disk for faster filtered queries.

```sql
-- Optimize with Z-Ordering on a frequently-filtered column
OPTIMIZE main.default.orders ZORDER BY (customer_id);
```

**Key term:** `OPTIMIZE` doesn't delete anything -- it writes *new*, better-organized files and
marks the old small files as no longer part of the table's current version, while keeping them
around (for time travel) until `VACUUM` physically removes them. Running `OPTIMIZE` alone doesn't
reclaim storage cost; it's `VACUUM` that does.
{: .important }

## `VACUUM`: physical deletion

```sql
VACUUM main.default.orders;              -- uses table's default retention (7 days)
VACUUM main.default.orders RETAIN 168 HOURS;  -- explicit retention window
```

`VACUUM` permanently deletes data files that are (a) no longer referenced by the table's current
version, and (b) older than the retention threshold. This is where files superseded by `OPTIMIZE`,
old `MERGE`/`UPDATE` versions, and deleted rows actually stop costing you storage.

**The retention floor matters.** Databricks defaults to a 7-day minimum retention and warns
strongly against going lower, because a `VACUUM` run with too-short retention can delete files a
still-running, long-lived job is actively reading from a still-valid earlier snapshot --
corrupting that job's read mid-flight, not just losing old history.
{: .important }

```sql
-- Only if you genuinely understand the risk -- not recommended for production tables
SET spark.databricks.delta.retentionDurationCheck.enabled = false;
VACUUM main.default.orders RETAIN 0 HOURS;
```

## `FULL` vs. `LITE` vacuum modes

`VACUUM` in `FULL` mode (the default) lists every file in the table's storage location to
determine what's unreferenced -- thorough, but can be slow on very large tables. `LITE` mode (a
newer, preview capability at time of writing) uses the transaction log itself to identify
deletable files instead of a full storage listing, faster for large tables but requires at least
one prior successful `FULL` vacuum to have already run.

## Soft-deleted data needs `REORG` first

Deletion vectors and column mapping (both used internally by `MERGE`/`UPDATE`/`DELETE` for
efficiency) can leave data **logically** deleted but not yet physically removed from files. Before
`VACUUM` can reclaim that space, run:

```sql
REORG TABLE main.default.orders APPLY (PURGE);
```

This physically rewrites files to remove soft-deleted data, after which a normal `VACUUM` can
reclaim the resulting orphaned files.

## A realistic maintenance schedule

```text
Nightly (after main pipeline runs):
  OPTIMIZE main.default.orders ZORDER BY (customer_id);

Weekly:
  REORG TABLE main.default.orders APPLY (PURGE);
  VACUUM main.default.orders;  -- default 7-day retention
```

Databricks recommends running `VACUUM` regularly rather than letting it accumulate -- a table left
unvacuumed for months can have a large amount of dead storage waiting to be reclaimed all at once,
which is itself a slower, more resource-intensive operation than routine incremental cleanup.

For the complete official reference on both modes, cluster sizing recommendations, and audit
logging for `VACUUM` operations, see
[VACUUM and data retention](https://docs.databricks.com/aws/en/delta/vacuum).

<!-- prevnext:start -->

---

| [&larr; Previous: Time Travel - querying historical versions]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/time-travel-querying-historical-versions/' | relative_url }}) | [Next: RESTORE and Rollback Strategies &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/restore-and-rollback-strategies/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
