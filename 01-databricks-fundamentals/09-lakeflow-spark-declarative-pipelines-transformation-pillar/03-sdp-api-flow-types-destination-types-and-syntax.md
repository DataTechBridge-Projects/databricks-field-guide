---
title: "SDP API - Flow types, Destination Types, and Syntax"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/sdp-api-flow-types-destination-types-and-syntax/
read_minutes: 24
---

# SDP API - Flow types, Destination Types, and Syntax
{: .no_toc }

*Estimated read: 24 min*

The complete syntax reference for everything the previous lecture introduced conceptually --
tables, views, materialized views, every flow type, and expectations, in both Python and SQL. This
is the lecture to bookmark and return to while building the hands-on bronze/silver/gold pipeline in
the lectures that follow.

## Streaming tables: Python and SQL

```python
import dlt as dp

@dp.table(
    name="bronze_orders",
    comment="Raw orders landed from cloud storage, minimally transformed"
)
def bronze_orders():
    return (spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("/Volumes/main/landing/orders/"))
```

```sql
CREATE OR REFRESH STREAMING TABLE bronze_orders
COMMENT "Raw orders landed from cloud storage, minimally transformed"
AS SELECT * FROM STREAM read_files(
  '/Volumes/main/landing/orders/',
  format => 'json'
);
```

Both produce the same destination type -- a streaming table, incrementally processed. SQL syntax
is often more concise for straightforward cases; Python is preferable once you need real
conditional logic, loops generating multiple similar tables, or integration with other Python
code.

## Materialized views: Python and SQL

```python
@dp.materialized_view(
    name="gold_daily_revenue",
    comment="Daily revenue aggregated by customer tier"
)
def gold_daily_revenue():
    return (spark.read.table("silver_orders")
        .groupBy("order_date")
        .agg({"order_total": "sum"}))
```

```sql
CREATE OR REFRESH MATERIALIZED VIEW gold_daily_revenue
COMMENT "Daily revenue aggregated by customer tier"
AS SELECT order_date, sum(order_total) AS total_revenue
FROM silver_orders
GROUP BY order_date;
```

## Append flows: adding data incrementally into an existing destination

Beyond the implicit append behavior of a simple streaming table read, `@dp.append_flow` lets
multiple, independent sources feed the *same* destination table -- useful when several upstream
sources logically belong in one bronze table:

```python
@dp.table(name="bronze_orders")
def bronze_orders_base():
    return spark.readStream.table("LIVE.orders_source_a")

@dp.append_flow(target="bronze_orders")
def orders_from_source_b():
    return spark.readStream.table("LIVE.orders_source_b")
```

Both flows append into the same `bronze_orders` destination, each independently, incrementally
tracked.

## AUTO CDC: SCD Type 1 and Type 2

**AUTO CDC** automates upsert logic from a change feed -- the declarative equivalent of the
`MERGE INTO` patterns from Section 5, without hand-writing the `MERGE` statement:

```python
dp.create_streaming_table("silver_customers")

dp.create_auto_cdc_flow(
    target="silver_customers",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by="updated_at",
    stored_as_scd_type=1   # overwrite in place -- only current state retained
)
```

```python
dp.create_auto_cdc_flow(
    target="silver_customers_history",
    source="bronze_customers_cdc",
    keys=["customer_id"],
    sequence_by="updated_at",
    stored_as_scd_type=2   # full history -- __START_AT / __END_AT columns track validity windows
)
```

| | SCD Type 1 | SCD Type 2 |
|---|---|---|
| Historical versions | Overwritten -- only current state | Preserved -- every version tracked |
| Extra columns | None | `__START_AT`, `__END_AT` |
| Use when | You only ever need current state | You need "what did this look like on date X" |

**Key term:** `sequence_by` is what tells AUTO CDC which incoming change is actually the *latest*
when multiple changes for the same key arrive close together or out of order -- almost always a
reliable timestamp or monotonic version column from the source, never wall-clock arrival time
(arrival order and true change order can differ, especially across retries or backfills).
{: .important }

## `AUTO CDC FROM SNAPSHOT`: when there's no change feed at all

For sources that only expose periodic full snapshots (no CDC feed available at all), a
snapshot-comparison variant detects changes by diffing consecutive snapshots:

```python
dp.create_auto_cdc_from_snapshot_flow(
    target="silver_products",
    source=lambda version: spark.read.table(f"bronze_products_snapshot_v{version}"),
    keys=["product_id"],
    stored_as_scd_type=2
)
```

This is the SDP-native answer to a source that genuinely can't provide row-level change events --
common for legacy systems exporting periodic full dumps rather than a real change stream.

## Expectations, in full

```python
@dp.table
@dp.expect("valid_email", "email IS NOT NULL AND email LIKE '%@%'")
@dp.expect_or_drop("valid_order_total", "order_total >= 0")
@dp.expect_or_fail("valid_order_id", "order_id IS NOT NULL")
def silver_orders():
    return spark.readStream.table("bronze_orders")
```

| Expectation | Violating rows | Use when |
|---|---|---|
| `expect` | Kept, flagged in pipeline metrics | Report-only, non-critical issues |
| `expect_or_drop` | Silently excluded from this destination | Quarantine-equivalent, moderate-severity issues |
| `expect_or_fail` | Entire pipeline run fails | Critical invariants that should never be violated |

This maps directly onto Section 7's report-only vs. quarantine strategies, plus a third tier
(`expect_or_fail`) for genuinely non-negotiable invariants -- a null primary key, for instance,
where continuing to process would be actively harmful rather than merely imprecise.

## Referencing other tables within the same pipeline

```python
@dp.table
def silver_orders():
    return spark.readStream.table("LIVE.bronze_orders")
    # or, in newer syntax without the LIVE. prefix, depending on pipeline configuration:
    # return spark.readStream.table("bronze_orders")
```

Tables within the same pipeline reference each other by name -- this is what lets SDP infer the
dependency graph automatically: `silver_orders` reading from `bronze_orders` tells the framework
`bronze_orders` must run first, without you writing any explicit ordering logic.

## Putting several concepts together

```python
import dlt as dp

@dp.table(comment="Raw orders, minimally transformed")
def bronze_orders():
    return (spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("/Volumes/main/landing/orders/"))

@dp.table(comment="Validated orders, bad rows quarantined")
@dp.expect_or_drop("valid_total", "order_total >= 0")
@dp.expect_or_fail("has_order_id", "order_id IS NOT NULL")
def silver_orders():
    return spark.readStream.table("bronze_orders")

@dp.materialized_view(comment="Daily revenue for the finance dashboard")
def gold_daily_revenue():
    return (spark.read.table("silver_orders")
        .groupBy("order_date")
        .agg({"order_total": "sum"}))
```

Three destinations, one dependency chain, inferred automatically from the table references -- no
checkpoint paths, no explicit `run bronze then silver then gold` ordering, no separate quality-
check code outside the table definition itself. This is the complete syntax vocabulary the next
three lectures build a real pipeline with.

<!-- prevnext:start -->

---

| [&larr; Previous: Core concepts - Pipelines, Flows, Destinations, and the Python API]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/core-concepts-pipelines-flows-destinations-and-the-python-api/' | relative_url }}) | [Next: Building the Bronze layer - Auto Loader as a Streaming Table &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-bronze-layer-auto-loader-as-a-streaming-table/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
