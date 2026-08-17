---
title: "Schema Enforcement and Schema Evolution"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 9
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/schema-enforcement-and-schema-evolution/
read_minutes: 14
---

# Schema Enforcement and Schema Evolution
{: .no_toc }

*Estimated read: 14 min*

Delta Lake's default behavior is strict: a write that doesn't match a table's schema **fails**,
loudly, at write time. This lecture covers why that default is a feature, and the deliberate,
explicit ways to change a table's schema when you actually need to -- rather than letting it drift
silently.

## Schema enforcement: the default

```python
# Table schema: order_id BIGINT, order_total DECIMAL(10,2)
df_bad = spark.createDataFrame([(1, "not-a-number")], ["order_id", "order_total"])
df_bad.write.format("delta").mode("append").saveAsTable("main.default.orders")
# Fails: schema mismatch -- write rejected before any data is committed
```

**Schema enforcement** means Delta Lake checks every write against the target table's schema, and
rejects anything that doesn't match -- wrong types, unexpected extra columns, missing required
columns. This is the same instinct as a warehouse's `NOT NULL` and type constraints, applied more
broadly: rather than silently accepting bad data and letting it corrupt something downstream, the
write simply fails, at the point where the mistake is easiest to diagnose.
{: .important }

## Explicit schema changes with `ALTER TABLE`

When a schema genuinely needs to change -- a new column, a wider type -- you say so explicitly:

```sql
ALTER TABLE main.default.orders ADD COLUMN discount_code STRING;
ALTER TABLE main.default.orders ADD COLUMN region STRING AFTER customer_id;
ALTER TABLE main.default.orders ALTER COLUMN order_total TYPE DECIMAL(12,2);
ALTER TABLE main.default.orders RENAME COLUMN order_total TO total_amount;
```

Adding columns, reordering them, widening a type (e.g. `DECIMAL(10,2)` to `DECIMAL(12,2)`,
`INT` to `BIGINT`), and renaming are all supported as metadata-only operations that don't require
rewriting existing data files.

## Automatic schema evolution during writes

For cases where you *want* incoming data to be allowed to introduce new columns automatically --
common for `MERGE`-based pipelines ingesting from a source that occasionally adds fields --
explicitly opt in per operation:

```sql
MERGE WITH SCHEMA EVOLUTION INTO main.default.orders AS target
USING staging_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

```python
(df.write
   .format("delta")
   .mode("append")
   .option("mergeSchema", "true")
   .saveAsTable("main.default.orders"))
```

**Key term:** schema evolution is **opt-in per statement**, not a global table setting you flip
once and forget -- Databricks explicitly recommends this over session-wide configuration, because
it keeps the decision "should this specific write be allowed to change the schema" visible in the
code doing the write, not buried in a session default someone else set.
{: .important }

## What schema evolution does and doesn't allow

| Allowed automatically with `mergeSchema`/`WITH SCHEMA EVOLUTION` | Requires explicit `ALTER TABLE` |
|---|---|
| New columns appearing in incoming data | Removing a column |
| | Renaming a column |
| | Changing an incompatible type (e.g. STRING to INT) |

Automatic evolution is deliberately conservative -- it only ever *adds*, never removes or
reinterprets. Anything destructive or ambiguous requires an explicit, reviewable `ALTER TABLE`
statement.

## The concurrency and streaming caveats

Two operational details worth knowing before you rely on this in production:

- **Schema updates conflict with concurrent writes.** An `ALTER TABLE` running at the same time as
  another write to the same table can fail or force a retry -- coordinate schema changes rather
  than assuming they're always safe to run anytime.
- **Schema changes terminate active streaming reads.** A stream reading from a table whose schema
  just changed doesn't silently adapt mid-flight -- it stops, and needs to be restarted to pick up
  the new schema. Plan schema changes for a maintenance window on tables with active downstream
  streams, not as a live, zero-impact operation.
{: .important }

## Why the default is strict, and when to relax it

Coming from a warehouse where a well-designed schema was itself the main data-quality control, this
should feel familiar rather than foreign: **enforcement is the default, evolution is the
exception you explicitly grant, one write at a time.** Silent, un-reviewed schema drift is exactly
the kind of failure mode a legacy warehouse's rigid schema protected you from by construction --
Delta Lake's default preserves that protection while still giving you a controlled way to evolve
when you genuinely need to.

For the complete official reference on both enforcement and evolution, including nested-field
(struct) schema changes and the `EXCEPT` clause for selective merge evolution, see
[Update Delta Lake table schema](https://docs.databricks.com/aws/en/delta/update-schema).

<!-- prevnext:start -->

---

| [&larr; Previous: DELETE, UPDATE and Idempotent writes]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/delete-update-and-idempotent-writes/' | relative_url }}) | [Next: Type Widening and the Variant Data Type &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/type-widening-and-the-variant-data-type/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
