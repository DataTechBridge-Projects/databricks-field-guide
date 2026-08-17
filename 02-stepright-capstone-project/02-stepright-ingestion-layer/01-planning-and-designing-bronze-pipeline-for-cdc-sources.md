---
title: "Planning and Designing Bronze Pipeline for CDC Sources"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 1
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-cdc-sources/
read_minutes: 7
---

# Planning and Designing Bronze Pipeline for CDC Sources
{: .no_toc }

*Estimated read: 7 min*

Section 1 left three raw change feeds sitting in `dev.raw_cdc`: `customers_changefeed`,
`orders_changefeed`, and `order_items_changefeed`. This lecture plans the bronze pipeline that
turns them into governed Delta tables before Lecture 2 writes a line of it.

## One pipeline, three tables

`orders`, `order_items`, and `customers` come from the same source system, deploy together, and
share the same lifecycle -- there's no reason to split them across three separate SDP pipelines.
`transformations/bronze_cdc.py` defines all three as streaming tables inside one pipeline, matching
the "one or a few tables per file, grouped logically" convention from Part 1, Section 9. Naming
follows the same convention throughout Part 2: `bronze_orders`, `bronze_order_items`,
`bronze_customers` -- source name, no suffix, since bronze here means "faithful copy of the feed,"
not "raw and unusable."

## Fixed schemas, not rescue mode

Part 1, Section 8's file-based ingestion patterns lean on Auto Loader's schema inference and
`rescue` mode -- appropriate when a source is a folder of files you don't control and schema drift
is expected. CDC sources are different: `dev.raw_cdc.*_changefeed` already comes from a managed
Lakeflow Connect connector reading a specific, known table structure in StepRight's operational
database. If that structure changes -- a column renamed, a type changed -- inferring around it and
rescuing the difference would silently absorb what is actually a breaking change upstream.

StepRight's CDC bronze tables use **explicit, fixed schemas** (`StructType` definitions) instead
of inference. A shape the pipeline doesn't recognize fails the read loudly, the same day it
happens, rather than landing a `_rescued_data` column three tables downstream that nobody notices
until silver breaks.
{: .important }

## What each fixed schema needs

Three fields matter for every CDC-sourced table, on top of its business columns:

| Field | Purpose |
|---|---|
| Primary key (`order_id`, `customer_id`, etc.) | What Section 3's AUTO CDC will key its SCD Type 2 merge on |
| A change timestamp (`updated_at`) | What AUTO CDC will sequence changes by, so a late-arriving out-of-order change doesn't overwrite a newer one |
| Business columns matching the source table | Everything Section 4's gold layer will eventually need -- adding a column later means a schema migration, so it's worth getting the shape right now |

## A shared read helper

All three tables do the same three things: read a stream from a `dev.raw_cdc` table, apply its
fixed schema, and add ingestion metadata (`_source_file` equivalent for CDC is `_ingested_at`,
since there's no file path). Rather than repeating that read logic three times, Lecture 2 factors
it into one helper function that every `@dp.table` definition calls with just a table name and
schema -- three tables, one place where the read logic can change.

## What's deferred to later lectures

This bronze layer stays deliberately thin: no deduplication, no business validation, no SCD
history. Deduplicating repeated CDC events and applying SCD Type 2 both belong to Section 3's
silver layer, via AUTO CDC -- bronze's only job is landing the feed faithfully. Structural quality
tagging (a null primary key, an unparseable timestamp) is real bronze-layer work, but it's split
into its own quality pipeline in Lectures 5 and 6, kept separate from the tables built in Lecture 2
so raw bronze always stays an untouched, replayable copy of what the source sent.

<!-- prevnext:start -->

---

| [&larr; Previous: StepRight - Ingestion Layer]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/' | relative_url }}) | [Next: Developing Bronze Pipeline for CDC Sources &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-pipeline-for-cdc-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

