---
title: "Building the Silver layer - AUTO CDC and SCD Type 2"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-silver-layer-auto-cdc-and-scd-type-2/
read_minutes: 13
---

# Building the Silver layer - AUTO CDC and SCD Type 2
{: .no_toc }

*Estimated read: 13 min*

Continuing directly from the previous lecture's bronze tables, this builds silver -- orders
validated with the report-only/quarantine pattern, and customers with full **SCD Type 2** history
via `AUTO CDC`, using explicit syntax rather than the abbreviated form.

## Silver orders: validation with expectations

```python
# transformations/silver_orders.py
import dlt as dp

@dp.table(
    name="silver_orders",
    comment="Validated orders -- critical failures dropped, minor issues flagged"
)
@dp.expect("reasonable_total", "order_total < 100000")
@dp.expect_or_drop("valid_total", "order_total >= 0")
@dp.expect_or_fail("has_order_id", "order_id IS NOT NULL")
def silver_orders():
    return (spark.readStream.table("bronze_orders")
        .select(
            "order_id", "customer_id", "order_total", "order_status", "order_date",
            "_ingested_at"
        ))
```

Three severities in one definition: `reasonable_total` flags (report-only) unusually large orders
worth a human glance without blocking anything; `valid_total` quarantines (drops from this
destination) negative totals; `has_order_id` fails the entire run if a core identity field is
missing -- exactly Section 7's report-only/quarantine framework, now with a third tier for
genuinely non-negotiable invariants.

## Silver customers: SCD Type 2 via `AUTO CDC`, explicit syntax

```python
# transformations/silver_customers.py
import dlt as dp

dp.create_streaming_table("silver_customers")

dp.create_auto_cdc_flow(
    target="silver_customers",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by="_commit_timestamp",
    apply_as_deletes="_change_type = 'delete'",
    except_column_list=["_change_type", "_commit_timestamp"],
    stored_as_scd_type=2
)
```

Walking through each argument:

- **`target`** -- the streaming table this flow populates (declared explicitly first with
  `create_streaming_table`, since `AUTO CDC` populates an existing destination rather than
  defining one inline).
- **`keys`** -- the natural key identifying a unique customer -- what `MERGE`'s `ON` clause would
  match on in the hand-written equivalent from Section 5.
- **`sequence_by`** -- the column establishing true change order, using the CDC feed's own commit
  timestamp (Section 8's `_commit_timestamp` from the managed CDC connector) rather than arrival
  time.
- **`apply_as_deletes`** -- an expression identifying delete events in the change feed, so AUTO CDC
  correctly closes out a customer's current SCD2 record rather than treating a delete as just
  another update.
- **`except_column_list`** -- columns from the source that shouldn't be carried into the target
  table (the CDC bookkeeping columns themselves, in this case).
- **`stored_as_scd_type=2`** -- full history, with `__START_AT`/`__END_AT` added automatically.

## Querying the SCD Type 2 result

```sql
-- Current customer state
SELECT * FROM dev.steprightproject.silver_customers WHERE __END_AT IS NULL;

-- A customer's full history
SELECT customer_id, email, __START_AT, __END_AT
FROM dev.steprightproject.silver_customers
WHERE customer_id = 12345
ORDER BY __START_AT;
```

`__END_AT IS NULL` identifies the current version of each customer -- the SDP-generated equivalent
of the `is_current = true` flag from the hand-written SCD2 pattern in Section 5's `MERGE INTO`
lecture, generated automatically rather than maintained by hand.

## Comparing to the hand-written Section 5 equivalent

| Hand-written `MERGE INTO` SCD2 (Section 5) | `AUTO CDC`, `stored_as_scd_type=2` |
|---|---|
| You write the full `MERGE` statement, matching, closing out old versions, inserting new | You declare keys, sequencing, and delete detection; the framework generates the equivalent logic |
| You manage `is_current`/`ended_at` columns yourself | `__START_AT`/`__END_AT` generated automatically |
| Deletes require separate handling logic | `apply_as_deletes` handles it declaratively |

This is the concrete payoff promised back in this section's first lecture: meaningfully less code
for the same correctness guarantee, once the pattern is common enough (SCD2 dimension maintenance)
to be worth a dedicated declarative API.

## What's built so far

Bronze (two sources) and silver (validated orders, full-history customers) are now both running as
a single, dependency-ordered pipeline. The next lecture adds gold-layer materialized views on top,
completing the full bronze-to-gold pipeline this section set out to build.

<!-- prevnext:start -->

---

| [&larr; Previous: Building the Bronze layer - Auto Loader as a Streaming Table]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-bronze-layer-auto-loader-as-a-streaming-table/' | relative_url }}) | [Next: Building the Gold layer - Materialized Views and Incremental Refresh &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-gold-layer-materialized-views-and-incremental-refresh/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
