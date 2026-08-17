---
title: "Incremental Ingestion, Schema Evolution and Bad Records in Production"
parent: "Lakeflow Connect - Ingestion Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/incremental-ingestion-schema-evolution-and-bad-records-in-production/
read_minutes: 12
---

# Incremental Ingestion, Schema Evolution and Bad Records in Production
{: .no_toc }

*Estimated read: 12 min*

Every ingestion pattern from this section works cleanly in a demo. This lecture covers what
actually goes wrong once a pipeline has been running against a real, messy source for months --
and the specific Auto Loader/Lakeflow Connect features built to handle it.

## Schema drift: the source changes without telling you

A source system adds a column, renames one, or occasionally sends a field with an unexpected type
-- with no warning, because you don't control the source. Auto Loader's schema evolution modes
govern what happens:

```python
.option("cloudFiles.schemaEvolutionMode", "addNewColumns")   # default: new columns added, stream restarts to pick them up
.option("cloudFiles.schemaEvolutionMode", "rescue")            # unexpected columns captured in a rescue column instead of failing
.option("cloudFiles.schemaEvolutionMode", "failOnNewColumns")  # stream stops entirely on schema change
.option("cloudFiles.schemaEvolutionMode", "none")               # new columns silently ignored
```

| Mode | Behavior | Use when |
|---|---|---|
| `addNewColumns` (default) | New columns added to schema; stream restarts | You want new fields captured automatically |
| `rescue` | Unexpected data captured in a `_rescued_data` column, stream keeps running | You want zero pipeline interruption, review rescued data separately |
| `failOnNewColumns` | Stream stops until you acknowledge the change | High-sensitivity pipelines where silent schema changes must be reviewed before they flow through |
| `none` | New columns dropped silently | Rare -- generally not recommended, since it hides real signal |

**Key term:** the **rescue column** pattern (`_rescued_data`) is worth calling out specifically --
rather than choosing between "fail the whole pipeline" and "silently drop unexpected data," it
captures anything that doesn't fit the expected schema into its own column, as raw JSON, so nothing
is lost and the main pipeline keeps running. Reviewing that column periodically is how you catch
drift without it ever blocking production.
{: .important }

## Bad records: malformed, not just differently-shaped

Distinct from schema drift, a **bad record** is one that's simply malformed -- invalid JSON, a
CSV row with the wrong number of fields, a value that can't be cast to the expected type at all:

```python
.option("cloudFiles.format", "json")
.option("badRecordsPath", "/Volumes/main/landing/orders_bad_records/")
```

Records that fail to parse are routed to `badRecordsPath` instead of failing the entire ingestion
run -- the file-ingestion equivalent of the silver-layer quarantine pattern from Section 7, applied
one layer earlier, at the point where data can't even be parsed into rows yet.

## A production ingestion pipeline, defensively configured

```python
raw_stream = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/Volumes/main/checkpoints/orders_schema/")
    .option("cloudFiles.schemaEvolutionMode", "rescue")
    .option("badRecordsPath", "/Volumes/main/landing/orders_bad_records/")
    .load("/Volumes/main/landing/orders/"))
```

This single configuration handles both failure modes this lecture covers: schema drift is
captured, not blocking; genuinely malformed records are routed aside, not blocking either. The
pipeline keeps running in both cases, with full visibility into what was rescued or rejected.

## Monitoring drift and bad records over time

```sql
-- How often is the rescue column non-null? A rising trend signals upstream drift worth investigating
SELECT date(_ingested_at), count(*) AS rescued_count
FROM bronze.orders
WHERE _rescued_data IS NOT NULL
GROUP BY date(_ingested_at)
ORDER BY 1 DESC;
```

Treating `_rescued_data` volume as a metric worth tracking over time -- not just a safety net you
never look at -- is what turns "the pipeline didn't break" into "we noticed the source changed
before it caused a downstream problem," which is the actual goal.
{: .important }

## Why this matters more from a legacy-ETL background

A legacy warehouse ETL job facing an unexpected source change often either failed loudly (blocking
the whole nightly batch) or silently truncated/dropped the offending rows, depending on how
defensively it was written. Auto Loader's rescue-column and bad-records patterns give you a third
option -- keep running, lose nothing, make the anomaly visible for review -- without requiring you
to hand-write that resilience into every single ingestion job yourself.

<!-- prevnext:start -->

---

| [&larr; Previous: File-Based Ingestion with Auto Loader]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/file-based-ingestion-with-auto-loader/' | relative_url }}) | [Next: Choosing the Right Ingestion Tool &rarr;]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/choosing-the-right-ingestion-tool/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
