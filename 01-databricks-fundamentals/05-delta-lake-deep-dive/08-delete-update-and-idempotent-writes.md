---
title: "DELETE, UPDATE and Idempotent writes"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/delete-update-and-idempotent-writes/
read_minutes: 11
---

# DELETE, UPDATE and Idempotent writes
{: .no_toc }

*Estimated read: 11 min*

`MERGE INTO` handles the common upsert case, but plain `DELETE` and `UPDATE` are still the right
tool for simpler operations -- and this lecture's second half covers something every production
pipeline needs regardless of which write method you use: making reruns safe.

## `DELETE`

```sql
DELETE FROM main.default.orders WHERE order_status = 'test_data';
```

Unlike a plain Parquet folder, `DELETE` against a Delta table is a genuine, ACID-safe operation --
readers never see a partially-deleted state, and the deletion is recorded in the transaction log
like any other commit (queryable via time travel, reversible via `RESTORE`, both from earlier
lectures in this section).

## `UPDATE`

```sql
UPDATE main.default.orders
SET order_status = 'cancelled'
WHERE order_id = 12345;
```

Straightforward row-level update, same ACID guarantees. For anything beyond "update rows matching
a simple condition" -- specifically, syncing a target table to match an incoming source -- prefer
`MERGE INTO` from the previous lecture; it's built for exactly that and avoids the read-then-write
race conditions a hand-rolled `UPDATE`-based sync can introduce under concurrent writes.

## Idempotency: the property that makes reruns safe

**Idempotent** means: running an operation twice produces the same result as running it once. This
matters enormously for pipelines, because pipelines *will* be rerun -- a job retries after a
transient failure, someone manually reruns a failed day's load, a scheduler double-triggers. If a
write isn't idempotent, a rerun corrupts data (duplicate rows from a naive `INSERT`, for example)
instead of harmlessly reproducing the same correct state.

```sql
-- NOT idempotent: rerunning this after a partial failure creates duplicates
INSERT INTO main.default.orders SELECT * FROM staging_orders;

-- Idempotent: rerunning this produces the same end state every time
MERGE INTO main.default.orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

**Key term:** `MERGE INTO` is idempotent *by construction* when keyed correctly -- matching rows
get updated (not duplicated) on a rerun, and unmatched rows insert exactly once. This is the single
biggest reason `MERGE` is preferred over plain `INSERT` for pipeline writes throughout this guide,
not just its upsert convenience.
{: .important }

## Idempotency for append-only streaming writes

Plain `INSERT`/append writes *can* still be made idempotent, using **`INSERT ... IF NOT EXISTS`**
patterns or, more commonly for streaming, Delta's built-in **idempotent write support** for
structured streaming, keyed by a transaction ID Spark tracks internally per micro-batch alongside
the checkpoint:

```python
(stream_df.writeStream
    .format("delta")
    .option("txnAppId", "orders_ingest_job")
    .option("txnVersion", batch_id)
    .outputMode("append")
    .table("main.default.orders"))
```

This tells Delta "this specific micro-batch, identified by this app ID and version, should only
ever be committed once" -- a retry of the same micro-batch after a partial failure is recognized
and skipped rather than reapplied.

## Why this matters more than it seems

In a legacy warehouse ETL job, idempotency was often achieved by convention and discipline --
truncate-and-reload patterns, careful watermark tracking, manual "has this batch already run"
checks. Delta gives you two structural mechanisms for it (`MERGE`'s match semantics, and
streaming's transaction-ID tracking) instead of requiring you to build that discipline by hand
every time. It's still your responsibility to key `MERGE` statements correctly and set transaction
IDs meaningfully -- Delta provides the mechanism, not automatic correctness.

<!-- prevnext:start -->

---

| [&larr; Previous: MERGE INTO - the Delta Upsert Engine]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/merge-into-the-delta-upsert-engine/' | relative_url }}) | [Next: Schema Enforcement and Schema Evolution &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/schema-enforcement-and-schema-evolution/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
