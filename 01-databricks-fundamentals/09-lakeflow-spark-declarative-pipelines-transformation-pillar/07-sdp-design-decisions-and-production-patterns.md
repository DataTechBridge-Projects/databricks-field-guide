---
title: "SDP Design Decisions and Production Patterns"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/sdp-design-decisions-and-production-patterns/
read_minutes: 8
---

# SDP Design Decisions and Production Patterns
{: .no_toc }

*Estimated read: 8 min*

A closing lecture on the judgment calls that separate a working demo pipeline from one you'd trust
in production -- pulling together everything built across this section.

## When to reach for SDP vs. manual Sections 5-8 code

SDP isn't strictly better in every case -- it's the right tool when:

- You're building a genuine Medallion pipeline with clear bronze -> silver -> gold dependencies.
- SCD Type 2 or other CDC-driven logic is involved -- `AUTO CDC` alone justifies SDP for many
  teams.
- You want the dependency graph, event log, and expectation metrics without building that
  observability yourself.

Manual, imperative code (Sections 5-8's approach) still has a place for:

- One-off scripts, ad hoc analysis, or genuinely simple single-table jobs where SDP's overhead
  isn't earning its keep.
- Highly custom orchestration logic that doesn't fit the declarative table-dependency model
  cleanly.

## One pipeline per logical unit, not one giant pipeline

A common early mistake is putting an entire organization's tables into one massive pipeline. This
makes the dependency graph unreadable and means an unrelated table's failure can block deployment
or monitoring clarity for tables that have nothing to do with it. Scope pipelines to a coherent
project or domain -- [Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}) uses this exact
discipline, with separate pipeline definitions per logical layer of the project.
{: .important }

## Expectations: pick severity deliberately, not by default

It's tempting to reach for `expect_or_fail` everywhere, out of an instinct that stricter is always
safer. In practice, an overly strict pipeline that fails entirely on a minor data quality blip
trains people to ignore failures (or worse, disable the pipeline) rather than actually investigate
them. Reserve `expect_or_fail` for genuinely non-negotiable invariants; use `expect_or_drop` for
real quality gates; use plain `expect` liberally for anything worth monitoring but not worth
blocking on.

## Testing pipeline logic before it's a pipeline

Because SDP table functions are ultimately plain Python functions returning DataFrames, the core
transformation logic inside them is unit-testable independent of the pipeline framework itself --
extract the actual transformation into a plain function, call it from the `@dp.table`-decorated
function, and test the plain function directly with `pytest` and a local Spark session. This is
exactly the pattern [Part 2's StepRight unit testing section]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/' | relative_url }})
builds out in full.

```python
def _compute_daily_revenue(orders_df, customers_df):
    # Plain, testable function -- no dlt/dp dependency
    return orders_df.join(customers_df, "customer_id").groupBy("order_date").sum("order_total")

@dp.materialized_view
def gold_daily_revenue():
    return _compute_daily_revenue(
        spark.read.table("silver_orders"),
        spark.read.table("silver_customers")
    )
```

## Naming and comments matter more than they seem to

Every table definition's `comment` argument shows up directly in the pipeline UI's dependency
graph and in Unity Catalog's table metadata -- a pipeline with clear, specific comments
("Validated orders, negative totals quarantined, missing order_id fails the run") is
self-documenting in a way a pipeline with no comments, or generic ones, simply isn't, months
later when someone else (or you) needs to understand what a table actually guarantees.

## Closing this section

This closes Lakeflow Declarative Pipelines. Data now flows bronze -> silver -> gold, declaratively,
with quality gates and full history where needed. The final section of Part 1, **Lakeflow Jobs**,
covers the third pillar -- how pipelines like the one you just built actually get **scheduled,
orchestrated, and monitored** in production, rather than run manually from the UI.

<!-- prevnext:start -->

---

| [&larr; Previous: Building the Gold layer - Materialized Views and Incremental Refresh]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-gold-layer-materialized-views-and-incremental-refresh/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
