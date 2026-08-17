---
title: "Planning and Designing Silver Pipeline for CDC Sources"
parent: "StepRight - Transformation Layers"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-cdc-sources/
read_minutes: 10
---

# Planning and Designing Silver Pipeline for CDC Sources
{: .no_toc }

*Estimated read: 10 min*

Lecture 1 designed twelve rules and a report-only/critical split. This lecture plans how that
split actually sits relative to `AUTO CDC`'s merge, and what each of the three CDC-sourced silver
tables needs by way of keys, sequencing, and delete handling before Lecture 3 writes the pipeline.

## The pipeline shape: tag, split, then merge

Three stages, in order, per table:

1. **Tag** -- read `bronze_<table>_valid` (Section 2's output, already structurally sound) and
   apply that table's business rules, producing `_dq_critical_failed` and `_dq_warnings` columns.
2. **Split** -- fan out exactly like Section 2, Lecture 6's quarantine pattern: rows where
   `_dq_critical_failed` is true go to `silver_<table>_quarantine`; everything else becomes the
   source stream `AUTO CDC` actually merges.
3. **Merge** -- `AUTO CDC` reads the critical-passed stream and merges it into
   `silver_<table>` with full SCD Type 2 history. `_dq_warnings` rides through the merge as an
   ordinary column, visible on every historical version of every row.

This is the same tag-don't-silently-drop discipline from Section 2, applied one layer up -- the
only new piece is that `AUTO CDC` needs a single clean stream to merge from, which is exactly why
the critical split has to happen *before* the merge rather than after.

## Keys and sequencing per table

| Table | `keys` | `sequence_by` | Why |
|---|---|---|---|
| `silver_customers` | `customer_id` | `updated_at` | Section 2's fixed CDC schema already carries `updated_at` as the change timestamp |
| `silver_orders` | `order_id` | `updated_at` | Same field, same reasoning -- order status changes (`PLACED` -> `SHIPPED` -> `DELIVERED`) are exactly what SCD2 history is for |
| `silver_order_items` | `order_item_id` | `updated_at` | A line item's `quantity` or `unit_price` can be corrected after the fact; history matters here too, not just on the parent order |

Using `updated_at` for all three keeps the sequencing story simple across the whole CDC silver
layer -- Section 2, Lecture 2 made sure every fixed CDC schema carries this column specifically so
this decision wouldn't need three different answers.

## Deletes: deliberately not handled

Part 1, Section 9's `AUTO CDC` customer example included `apply_as_deletes`, because that source
had genuine delete events in its change feed. StepRight's operational order system doesn't hard-
delete customers, orders, or order items -- a cancelled order becomes `status = 'CANCELLED'`, not
a removed row. All three `AUTO CDC` flows in Lecture 3 omit `apply_as_deletes` entirely. If
StepRight's source system ever added real deletes (a GDPR erasure request, for instance), that
would be a deliberate, reviewed change to this pipeline, not something to build defensively for
today against a case that can't currently happen.

## `except_column_list`: dropping the tagging scaffolding

`_dq_critical_failed` did its job once the split happened -- it has no reason to persist into the
final SCD2 table and would only confuse anyone reading `silver_orders` later wondering why a
column named for failure is present on every row that, by definition, passed. Each `AUTO CDC` flow
in Lecture 3 excludes it via `except_column_list`, while explicitly keeping `_dq_warnings` -- the
one tagging column meant to survive into the final table.

## One pipeline, three flows

Following the same grouping logic as Section 2's bronze CDC pipeline, all three tag-split-merge
flows live in one `transformations/silver_cdc.py`, deployed as part of the same pipeline as
bronze -- SDP's dependency inference means silver's read of `bronze_orders_valid` and the rest
automatically orders these tables after Section 2's bronze tables in the pipeline graph, with no
manual sequencing required.

<!-- prevnext:start -->

---

| [&larr; Previous: Design the Silver Layer Data Quality Rules and Approach]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/design-the-silver-layer-data-quality-rules-and-approach/' | relative_url }}) | [Next: Developing Silver Pipeline for AUTO CDC Sources &rarr;]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/developing-silver-pipeline-for-auto-cdc-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

