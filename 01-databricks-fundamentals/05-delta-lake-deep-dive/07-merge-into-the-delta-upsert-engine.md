---
title: "MERGE INTO - the Delta Upsert Engine"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/merge-into-the-delta-upsert-engine/
read_minutes: 17
---

# MERGE INTO - the Delta Upsert Engine
{: .no_toc }

*Estimated read: 17 min*

If you've written an Oracle `MERGE` or a SQL Server `MERGE`/hand-rolled upsert (update-if-exists,
insert-if-not) in a legacy ETL job, Delta Lake's `MERGE INTO` will feel immediately familiar --
same problem, same general shape, running against a Lakehouse table instead. This is one of the
highest-leverage statements in this entire guide; almost every silver-layer pipeline in Part 2
uses it.

## Basic upsert

```sql
MERGE INTO main.default.orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN
  UPDATE SET *
WHEN NOT MATCHED THEN
  INSERT *;
```

This is the pattern: match rows between a target table and an incoming source (a staging table, a
streaming micro-batch, a CDC feed) on a key, update matched rows, insert unmatched ones. `UPDATE
SET *` and `INSERT *` update/insert every column by name-matching -- explicit column lists are
also supported and often clearer in production code.

## The full clause set

```sql
MERGE INTO main.default.orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED AND source.is_deleted = true THEN
  DELETE
WHEN MATCHED AND source.updated_at > target.updated_at THEN
  UPDATE SET
    order_total  = source.order_total,
    order_status = source.order_status,
    updated_at   = source.updated_at
WHEN NOT MATCHED THEN
  INSERT (order_id, customer_id, order_total, order_status, updated_at)
  VALUES (source.order_id, source.customer_id, source.order_total, source.order_status, source.updated_at)
WHEN NOT MATCHED BY SOURCE THEN
  UPDATE SET target.order_status = 'archived';
```

| Clause | Meaning |
|---|---|
| `WHEN MATCHED ... THEN UPDATE` | Row exists in both -- update it (optionally conditioned) |
| `WHEN MATCHED ... THEN DELETE` | Row exists in both, but source signals deletion -- remove it |
| `WHEN NOT MATCHED THEN INSERT` | Row exists only in source -- it's new, insert it |
| `WHEN NOT MATCHED BY SOURCE` | Row exists only in target -- source no longer has it |

**Key term:** `WHEN NOT MATCHED BY SOURCE` is the clause most people coming from a warehouse
`MERGE` haven't used as often -- it handles rows present in the target but *absent* from the
source, useful for soft-deleting or archiving records a source system has since removed, rather
than only ever adding/updating.

## The single-match constraint

A `MERGE` requires that **at most one row** in the source match a given target row (or vice versa
for insert-side matching) -- if your source has duplicate keys, `MERGE` fails with an error rather
than silently picking one arbitrarily or applying both. This is a deliberate correctness guarantee,
not a bug -- deduplicate the source *before* the `MERGE`, not by hoping the engine picks
reasonably.
{: .important }

```sql
-- Deduplicate source before merging, keeping the latest by updated_at
CREATE OR REPLACE TEMP VIEW deduped_source AS
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) AS rn
  FROM staging_orders
) WHERE rn = 1;
```

## SCD Type 2 with `MERGE INTO`

A common pattern this guide revisits directly in Section 9 (Lakeflow Declarative Pipelines) and
throughout Part 2: maintaining a **slowly changing dimension Type 2** history (every change to a
row creates a new version with `started_at`/`ended_at` timestamps, rather than overwriting in
place) using `MERGE`:

```sql
MERGE INTO customers_scd2 AS target
USING (
  SELECT *, current_timestamp() AS effective_ts FROM staging_customers
) AS source
ON target.customer_id = source.customer_id AND target.is_current = true
WHEN MATCHED AND target.email != source.email THEN
  UPDATE SET target.is_current = false, target.ended_at = source.effective_ts
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, is_current, started_at, ended_at)
  VALUES (source.customer_id, source.email, true, source.effective_ts, NULL);
```

This single statement only closes out changed rows -- a second `MERGE` (or the `AUTO CDC` flow API
covered in Section 9, which automates this exact pattern) inserts the new current version.

## Comparing to your legacy MERGE

| Legacy warehouse `MERGE` | Delta `MERGE INTO` |
|---|---|
| Runs against a single-engine, ACID-native table | Runs against Parquet + transaction log, same ACID guarantees |
| Often locks the target table during the operation | Optimistic concurrency -- conflicting concurrent writes retry or fail cleanly, not blocking readers |
| Source is usually another table in the same engine | Source can be a table, a view, a streaming micro-batch, or a DataFrame |

The mental model transfers almost entirely -- the syntax differences are smaller than the
underlying execution model differences, and those differences (no table locking, works against
streaming micro-batches) are generally advantages once you're used to them.

For the full official syntax reference, including Python/Scala DeltaTable API equivalents beyond
the SQL shown here, see [Delta Lake MERGE INTO](https://docs.databricks.com/aws/en/delta/merge).

<!-- prevnext:start -->

---

| [&larr; Previous: RESTORE and Rollback Strategies]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/restore-and-rollback-strategies/' | relative_url }}) | [Next: DELETE, UPDATE and Idempotent writes &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/delete-update-and-idempotent-writes/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
