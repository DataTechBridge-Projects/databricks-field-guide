---
title: "Planning and Designing Silver Pipeline for File Sources"
parent: "StepRight - Transformation Layers"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-file-sources/
read_minutes: 2
---

# Planning and Designing Silver Pipeline for File Sources
{: .no_toc }

*Estimated read: 2 min*

`bronze_products_valid` is structurally sound but not yet silver-ready: batch zero and every later
catalog update land as an append-only stream of files, so the same `product_id` can appear more
than once as merchandising reissues an update. This lecture plans the (much smaller) silver step
that fixes that -- no `AUTO CDC`, because there's no change feed with delete semantics to merge,
just a batch of files where the newest one wins.

## Latest-wins, not SCD Type 2

`orders`, `customers`, and `order_items` needed full history because Section 4's gold layer cares
about *change over time* (a customer's loyalty tier last quarter, an order's status at each stage).
The product catalog doesn't carry that same requirement here -- gold's product performance report
(Section 4) wants the **current** list price and category, not a history of every past price.
`silver_products` keeps only the newest row per `product_id`, using `_ingested_at` (Section 2,
Lecture 4's file-bronze metadata column) to break ties.

## Validity, not referential checks

`silver_products` has no foreign keys to resolve -- it *is* the table every other silver rule in
this section resolves against. Its own business rules are simple validity checks: `list_price`
must be positive, `category` must be one of StepRight's known categories. Neither rises to
"critical" the way `silver_order_items`' rules did -- an invalid category still identifies a real
product worth showing in a catalog listing, so both stay report-only, tagged with the same
`_dq_warnings` pattern rather than a critical/quarantine split this table doesn't need.

## Why `inventory`, `clickstream`, and `fulfillment` stop at bronze

These three don't get a `silver_*` table in this section. Section 4's clickstream funnel and
fulfillment health reports need the raw event grain -- one row per click, one row per shipment
status update -- not a deduplicated "current" view the way `products` needed one. Building a
silver layer for them now would mean guessing at a shape before knowing exactly what each gold
report needs; Section 4 reads `bronze_inventory_valid`, `bronze_clickstream_valid`, and
`bronze_fulfillment_valid` directly and applies whatever light transformation each specific report
requires, in place, rather than through a speculative intermediate layer.

Lecture 5 builds `silver_products` against this plan.

<!-- prevnext:start -->

---

| [&larr; Previous: Developing Silver Pipeline for AUTO CDC Sources]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/developing-silver-pipeline-for-auto-cdc-sources/' | relative_url }}) | [Next: Developing Silver SDP Pipeline for File Sources &rarr;]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/developing-silver-sdp-pipeline-for-file-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

