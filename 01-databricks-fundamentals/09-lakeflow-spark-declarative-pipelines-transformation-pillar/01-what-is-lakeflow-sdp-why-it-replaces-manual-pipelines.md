---
title: "What is Lakeflow SDP - Why it Replaces Manual Pipelines"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 1
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/what-is-lakeflow-sdp-why-it-replaces-manual-pipelines/
read_minutes: 8
---

# What is Lakeflow SDP - Why it Replaces Manual Pipelines
{: .no_toc }

*Estimated read: 8 min*

Sections 5-8 taught you to hand-write every piece of a bronze-to-gold pipeline: `MERGE`
statements, streaming reads/writes, checkpoints, Auto Loader configuration. **Lakeflow Declarative
Pipelines** (also called **Spark Declarative Pipelines**, or "SDP") is the framework that lets you
*declare* the pipeline instead -- describing what each table should contain, and letting
Databricks handle dependency ordering, checkpoint management, and incremental execution.

## Imperative vs. declarative, concretely

Everything you've written so far this guide has been **imperative**: explicit `readStream`,
explicit `writeStream`, explicit checkpoint paths, explicit ordering of which cell runs before
which. **Declarative** pipelines invert this -- you describe the *destination table's* logic, and
the framework figures out execution order, incremental processing, and dependency management from
the dependency graph implied by your table definitions referencing each other.

```python
# Imperative (what you've written so far)
stream_df = spark.readStream.format("delta").table("bronze.orders")
(stream_df.writeStream
    .option("checkpointLocation", "/checkpoints/silver_orders/")
    .table("silver.orders"))

# Declarative (SDP) -- the framework manages checkpoints and execution
@dp.table
def silver_orders():
    return spark.readStream.table("bronze_orders")
```

**Key term:** in SDP, you never write a `checkpointLocation` yourself -- the framework manages
checkpoints, dependency ordering, and retry/restart behavior for every table in the pipeline,
inferred from the fact that `silver_orders` reads from `bronze_orders`.
{: .important }

## What SDP manages that you were doing by hand

| Manual (Sections 5-8) | SDP |
|---|---|
| Explicit checkpoint paths per stream | Managed automatically |
| Manual dependency ordering across notebooks | Inferred from table read/write references |
| Hand-written data quality checks | Declarative `@dp.expect` constraints |
| Separate CDC `MERGE` logic per table | `AUTO CDC` API (Section 5's coverage, at "here's the expression" level) |
| Manually wiring bronze -> silver -> gold execution | Automatic, dependency-graph-driven execution |

## Why this matters for reducing boilerplate

A hand-written Medallion pipeline with, say, five bronze tables, eight silver tables, and four
gold tables means roughly seventeen separate streaming jobs to write, checkpoint, and keep
synchronized by hand -- and every added data quality check is more manual code. SDP collapses that
into seventeen **table definitions**, each describing its own logic, with the framework handling
everything about *how* those definitions actually execute reliably.

## Where SDP fits relative to what you already know

This isn't a replacement for the Delta Lake concepts from Section 5 -- `MERGE`, schema
enforcement, time travel all still apply underneath. SDP is a **higher-level authoring layer** on
top of them: you're still ultimately producing Delta tables with all the same guarantees, just
described declaratively instead of imperatively assembled cell by cell.

## What this section builds

The rest of this section works through SDP's core concepts (pipelines, flows, destinations),
its API syntax in depth, and then builds a full bronze -> silver -> gold pipeline hands-on --
directly the shape [Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}) uses for its own
production pipelines.

<!-- prevnext:start -->

---

| [&larr; Previous: Lakeflow Spark Declarative Pipelines - Transformation Pillar]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/' | relative_url }}) | [Next: Core concepts - Pipelines, Flows, Destinations, and the Python API &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/core-concepts-pipelines-flows-destinations-and-the-python-api/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
