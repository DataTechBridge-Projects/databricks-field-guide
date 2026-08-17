---
title: "Core concepts - Pipelines, Flows, Destinations, and the Python API"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/core-concepts-pipelines-flows-destinations-and-the-python-api/
read_minutes: 9
---

# Core concepts - Pipelines, Flows, Destinations, and the Python API
{: .no_toc }

*Estimated read: 9 min*

Three vocabulary terms -- **pipeline**, **flow**, and **destination** -- and the shape of the
Python API you'll use throughout the rest of this section.

## Pipeline

A **pipeline** is the top-level unit -- a collection of table definitions (and the flows that
populate them) that Databricks deploys and runs together as a single dependency graph. One
pipeline typically covers a related set of bronze/silver/gold tables, not your entire
organization's data estate in one giant pipeline.

## Destinations: streaming tables and materialized views

Every table an SDP pipeline produces is one of two **destination** types:

- **Streaming table** -- backed by incremental, append-oriented processing (the streaming API
  from Section 5, managed automatically). Right for bronze and most silver tables, where new data
  arrives continuously and should be processed incrementally.
- **Materialized view** -- recomputed (fully or incrementally, where possible) on each pipeline
  run, right for gold-layer aggregations where the output depends on the *current full state* of
  its inputs, not just new rows since last run.

```python
import dlt as dp

@dp.table
def bronze_orders():
    return spark.readStream.format("cloudFiles").option("cloudFiles.format", "json").load("/Volumes/main/landing/orders/")

@dp.materialized_view
def gold_daily_revenue():
    return spark.read.table("silver_orders").groupBy("order_date").sum("order_total")
```

**Key term:** the choice between streaming table and materialized view is the SDP-native version
of the batch-vs-streaming and view-vs-table decisions from Sections 5 and 7 -- same underlying
tradeoff, now expressed as a decorator choice rather than separate manually-written code paths.
{: .important }

## Flows: the four kinds

A **flow** is the specific logic populating a destination. Four flow types cover essentially every
case:

| Flow type | Behavior |
|---|---|
| **Append flow** | Straightforward incremental append -- new rows added, nothing changed or removed |
| **AUTO CDC flow** | Automated upsert/SCD logic from a change feed -- covered in depth in the silver-layer lecture |
| **Materialized view flow** | Recomputation logic for a materialized view destination |
| **Update flow** | General update logic beyond simple append, for cases append alone doesn't cover |

```python
@dp.table
def bronze_orders():
    return spark.readStream.format("cloudFiles").option("cloudFiles.format", "json").load("/Volumes/main/landing/orders/")
    # implicitly an append flow -- new files, new rows, appended
```

## Data quality expectations

```python
@dp.table
@dp.expect_or_drop("valid_order_total", "order_total >= 0")
@dp.expect("recent_order", "order_date >= current_date() - INTERVAL 2 YEARS")
def silver_orders():
    return spark.readStream.table("bronze_orders")
```

**`expect`** (report-only, matching Section 7's report-only pattern) and **`expect_or_drop`**
(quarantine-equivalent -- violating rows are dropped from this destination) map directly onto the
Medallion data quality strategies from Section 7, expressed as declarative constraints attached
directly to the table definition rather than separate filtering logic you'd write by hand.

## The Python API shape, summarized

```python
import dlt as dp

@dp.table(name="...", comment="...")
@dp.expect_or_drop("rule_name", "sql_condition")
def function_name():
    return <a DataFrame — batch or streaming read>
```

Every SDP table definition follows this shape: a decorator declaring the destination and any
quality expectations, wrapping a plain Python function that returns a DataFrame. The function body
is ordinary Spark code you already know from Sections 5-8 -- SDP's contribution is everything
*around* that function, not a new way of writing transformations themselves.

The next lecture goes deeper into the full syntax across flow and destination types before this
section moves into building bronze, silver, and gold hands-on.

<!-- prevnext:start -->

---

| [&larr; Previous: What is Lakeflow SDP - Why it Replaces Manual Pipelines]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/what-is-lakeflow-sdp-why-it-replaces-manual-pipelines/' | relative_url }}) | [Next: SDP API - Flow types, Destination Types, and Syntax &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/sdp-api-flow-types-destination-types-and-syntax/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
