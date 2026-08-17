---
title: "Design Source Data Quality Monitoring in Bronze Layer"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/design-source-data-quality-monitoring-in-bronze-layer/
read_minutes: 9
---

# Design Source Data Quality Monitoring in Bronze Layer
{: .no_toc }

*Estimated read: 9 min*

Seven bronze tables now exist, and Section 1's seed generator deliberately put bad rows in some of
them -- orphaned foreign keys, a negative `unit_price`, a null `email`. This lecture designs how
bronze tells the difference between a row that's structurally usable and one that isn't, without
touching the raw tables Lectures 2 and 4 just built.

## Bronze stays bronze

The single most important design decision here: **raw bronze tables are never modified,
filtered, or dropped from.** `bronze_orders`, `bronze_products`, and the rest stay exactly what
Lecture 2 and Lecture 4 built -- a faithful, replayable copy of what the source sent, bad rows and
all. That's what makes bronze useful as a debugging and backfill source later; a bronze table
that's already had "bad" rows quietly removed can't answer "what did the source actually send us
on March 3rd."

Quality tagging happens **downstream** of raw bronze, in a separate set of tables this lecture
calls the **bronze quality layer** -- read raw bronze, tag every row, write the tagged result
somewhere else. Silver, built in Section 3, reads from the quality layer's valid output, never
from raw bronze directly.
{: .important }

## Structural checks, not business rules

Part 1, Section 7 drew this line for bronze in general; it applies just as strictly here. A
structural check asks "is this row usable at all" -- a null primary key, a foreign key that
doesn't resolve, a price that's negative. A business rule asks "is this row *correct*" -- a
discount that's unusually high for this customer's tier, an order total that doesn't match its
line items. Business rules are Section 3's job, applied against the twelve silver-layer rules
across four categories that section designs. Bronze's quality layer only checks structure:

| Table | Structural checks |
|---|---|
| `bronze_customers` | `customer_id` not null |
| `bronze_orders` | `order_id` not null, `customer_id` resolves to a known customer |
| `bronze_order_items` | `order_item_id` not null, `order_id` resolves, `product_id` resolves, `unit_price` >= 0, `quantity` > 0 |
| `bronze_products` | `product_id` not null, `list_price` not null |

Referential checks (`customer_id` resolves, `product_id` resolves) need a lookup against the
*other* bronze table, not just a null check within the same row -- which is exactly the case
Section 1, Lecture 4's ~1-2% deliberately orphaned foreign keys were generated to exercise.

## Tag, don't drop

Every row gets two new columns rather than being silently dropped when it fails a check:

- **`_dq_valid`** (`boolean`) -- did this row pass every structural check for its table.
- **`_dq_failed_rules`** (`array<string>`) -- which named check(s) it failed, empty if none.

Tagging instead of dropping means a failed row is still visible, countable, and explainable --
"how many order items failed last night, and why" is a one-line query against
`_dq_failed_rules`, not a mystery about rows that silently vanished between bronze and silver.

## Where valid and quarantined rows go

Each bronze table's quality layer produces two outputs from Lecture 6's implementation:

- **`bronze_<table>_valid`** -- rows where `_dq_valid = true`. This is what Section 3's silver
  pipeline actually reads from.
- **`bronze_<table>_quarantine`** -- rows where `_dq_valid = false`, kept for investigation,
  reprocessing once a source issue is fixed, or inclusion in Section 7's data quality dashboard.

## Why a separate quality pipeline

Keeping quality tagging in its own transformation files, downstream of the raw bronze tables built
in Lectures 2 and 4, means the tagging rules can change -- a new structural check added, a
threshold adjusted -- without touching or redeploying the ingestion pipelines themselves. It also
means the raw bronze tables can be reprocessed against updated quality rules at any time, since
they were never coupled to a specific version of the quality logic in the first place. Lecture 6
builds this pipeline for real.

<!-- prevnext:start -->

---

| [&larr; Previous: Developing Bronze SDP Pipeline for File Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-sdp-pipeline-for-file-sources/' | relative_url }}) | [Next: Developing Quarantine Pattern in Bronze Layer &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-quarantine-pattern-in-bronze-layer/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

