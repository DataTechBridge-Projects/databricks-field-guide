---
title: "Planning and Designing Bronze Pipeline for File Sources"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-file-sources/
read_minutes: 3
---

# Planning and Designing Bronze Pipeline for File Sources
{: .no_toc }

*Estimated read: 3 min*

Four sources land as files in `dev.step_right.landing`: `products/`, `inventory/`,
`clickstream/`, and `fulfillment/`. This lecture plans their bronze pipeline; Lecture 4 builds it.

## Same source shape, opposite schema discipline from Lecture 1

These are exactly the kind of source Auto Loader was built for: files dropped by an external
system StepRight's engineering team doesn't control, on a schedule nobody guarantees. That's the
mirror image of Lecture 1's CDC design decision -- where CDC bronze uses fixed schemas because
drift should fail loudly, file-based bronze uses **schema inference with `rescue` mode**, the same
pattern from Part 1, Sections 8 and 9. If merchandising adds a new column to the product catalog
export next quarter, this pipeline should keep running and rescue the new column, not break a
scheduled job at 2 a.m.

## One table per source, same helper pattern

`bronze_products`, `bronze_inventory`, `bronze_clickstream`, `bronze_fulfillment` -- four streaming
tables, one per landing subfolder, following the same naming convention as the CDC tables. All
four read with the same `cloudFiles` options (format, schema location, `rescue` mode) and add the
same two metadata columns (`_source_file`, `_ingested_at`) that Part 1, Section 9's bronze example
used -- which means, just like Lecture 1's CDC helper, this is a case for a small factory function
rather than four near-identical `@dp.table` definitions.

## Format and cadence per source

| Source | Format | Cadence |
|---|---|---|
| `products/` | JSON | Irregular -- merchandising pushes catalog updates as needed |
| `inventory/` | JSON | Daily snapshot from the 3PL |
| `clickstream/` | JSON | Frequent, small batches throughout the day |
| `fulfillment/` | JSON | Per-shipment, as the carrier integration confirms status |

All four happen to be JSON in this project, which keeps the factory function in Lecture 4 simple --
a real environment with a mix of CSV exports and JSON event streams would need the format passed
alongside each source name, which the same factory function handles just as easily.

## Scope for Lecture 4

Building and fully verifying all four tables would repeat the same pattern four times with little
new to learn on tables two through four. Lecture 4 builds `bronze_products` in full detail, then
generates the remaining three from the same factory function -- the code that creates one table
correctly is the code that creates all four correctly, which is the entire argument for writing it
as a factory in the first place rather than four hand-copied functions.

<!-- prevnext:start -->

---

| [&larr; Previous: Developing Bronze Pipeline for CDC Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-pipeline-for-cdc-sources/' | relative_url }}) | [Next: Developing Bronze SDP Pipeline for File Sources &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-sdp-pipeline-for-file-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

