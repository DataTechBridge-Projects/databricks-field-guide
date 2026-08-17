---
title: "Time Travel - querying historical versions"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 4
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/time-travel-querying-historical-versions/
read_minutes: 10
---

# Time Travel - querying historical versions
{: .no_toc }

*Estimated read: 10 min*

**Time travel** is querying a Delta table as it existed at a prior point -- a direct, practical
payoff of the transaction log from the first lecture in this section. If you've ever wished you
could query a warehouse table "as of yesterday morning, before that bad load ran," this is exactly
that capability, built in rather than requiring a manual snapshot strategy.

## Querying by version

```sql
SELECT * FROM main.default.orders VERSION AS OF 42;
```

```python
df = spark.read.format("delta").option("versionAsOf", 42).table("main.default.orders")
```

Every commit to a Delta table increments its **version number** -- an integer you can see directly
via `DESCRIBE HISTORY`. Querying `VERSION AS OF` a specific number returns the table exactly as it
looked immediately after that commit.

## Querying by timestamp

```sql
SELECT * FROM main.default.orders TIMESTAMP AS OF '2026-08-01T00:00:00.000Z';
```

```python
df = spark.read.format("delta").option("timestampAsOf", "2026-08-01").table("main.default.orders")
```

Same idea, addressed by wall-clock time instead of version number -- often more natural for "what
did this look like before yesterday's bad run," where you know roughly *when* something went
wrong but not the exact version number.

## Inspecting the full history

```sql
DESCRIBE HISTORY main.default.orders;
```

Returns every commit -- version, timestamp, operation type (`WRITE`, `MERGE`, `DELETE`,
`OPTIMIZE`...), and the user or job that made it. This is your **audit trail**, directly comparable
to a warehouse's change-data-capture log, except it's native to the table rather than a bolted-on
CDC feature you had to explicitly enable.

## Real uses beyond curiosity

- **Auditing.** "Who changed this row, and when" -- answerable directly from `DESCRIBE HISTORY`
  without a separate audit table you had to build and maintain.
- **Debugging a bad pipeline run.** Compare `VERSION AS OF` the run before a suspected bug against
  the current state to isolate exactly what a specific job changed.
- **Reproducing an analysis.** A report generated last month can be re-run against the table state
  *as it was last month*, even if the table has since changed -- genuinely difficult to do
  reliably against a mutable warehouse table without time travel.
- **Manual rollback**, covered as its own dedicated pattern (with `RESTORE`) in a later lecture in
  this section.

## The retention limits that make this not-infinite

Time travel isn't free or unlimited -- two retention settings govern how far back you can actually
go:

- **`delta.deletedFileRetentionDuration`** (default 7 days) -- how long *data files* superseded by
  newer writes are kept before becoming eligible for physical deletion.
- **`delta.logRetentionDuration`** (default 30 days) -- how long transaction *log* entries
  themselves are kept.

```sql
ALTER TABLE main.default.orders
SET TBLPROPERTIES (
  'delta.deletedFileRetentionDuration' = '30 days',
  'delta.logRetentionDuration' = '60 days'
);
```

**Key term:** running `VACUUM` (next lecture) physically deletes data files older than the
retention window -- once that happens, time travel to versions relying on those files is
permanently impossible, no matter what the log retention setting says. Retention and `VACUUM` are
two halves of the same tradeoff: keep more history for auditing/rollback, or reclaim storage cost
sooner. Extending retention without a real audit/compliance need just accumulates storage cost for
history you'll never query.
{: .important }

For the complete official reference -- including exact syntax variations and Delta Runtime version
differences -- see
[Work with Delta Lake table history](https://docs.databricks.com/aws/en/delta/history).

<!-- prevnext:start -->

---

| [&larr; Previous: Reading and writing Delta - batch and streaming]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/reading-and-writing-delta-batch-and-streaming/' | relative_url }}) | [Next: OPTIMIZE, VACUUM and Data Retention &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/optimize-vacuum-and-data-retention/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
