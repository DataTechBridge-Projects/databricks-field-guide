---
title: "Test Data Strategy - Planning your Test Data Preparation"
parent: "StepRight - Project Foundation"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/01-stepright-project-foundation/test-data-strategy-planning-your-test-data-preparation/
read_minutes: 7
---

# Test Data Strategy - Planning your Test Data Preparation
{: .no_toc }

*Estimated read: 7 min*

There's no real StepRight production database to connect to -- this project needs a test data
strategy before it needs a single line of pipeline code, because every lecture from Section 2
onward assumes there's something real sitting in the landing volume and the CDC feed to read.

## What "real enough" means here

Generated data only earns its keep if it's structurally honest about the problems a production
source actually has. A flat, perfectly clean dataset would let every bronze and silver pipeline in
Sections 2 and 3 pass on the first try -- which defeats the purpose of building a quarantine
pattern and twelve silver-layer data quality rules in the first place. StepRight's test data needs
to include, deliberately:

- **Referential integrity that's mostly, not perfectly, intact** -- a small percentage of
  `order_items` rows referencing a `product_id` that doesn't exist in the catalog yet (a late
  or missing catalog update), and a small percentage of `orders` referencing a `customer_id` not
  yet present in the CDC feed (an ordering/timing problem CDC pipelines have to tolerate).
- **Out-of-range and malformed values** -- negative or absurd `unit_price` values, `discount_pct`
  values above 100%, null email addresses on a fraction of customer records.
- **Duplicate events** -- the same CDC change row appearing twice, exercising the idempotency
  Section 3's AUTO CDC handling depends on.
- **Realistic skew** -- most customers place one or two orders; a small number are frequent
  buyers; a handful of products account for a disproportionate share of order volume. Flat,
  uniform distributions make gold-layer aggregations in Section 4 look correct even when a
  join is silently dropping rows.

## Batches, not one static dump

A single one-time load can't exercise incremental behavior -- Auto Loader's exactly-once file
tracking, AUTO CDC's out-of-order change handling, and a scheduled Lakeflow Job all only prove
themselves across multiple runs. The generator built in Lecture 5 produces data in **numbered
batches**:

| Batch | Purpose |
|---|---|
| Batch zero | The initial seed -- enough customers, products, and historical orders to make Section 2's first bronze pipeline run meaningful on day one |
| Batch one+ | Smaller incremental batches -- new orders, a handful of customer updates (address change, loyalty tier upgrade), a product catalog delta -- generated and loaded on demand in later sections to prove incremental ingestion and SCD Type 2 history actually work |

This mirrors how a real CDC feed and real file drops behave: an initial full load, then a steady
trickle of changes. Testing only against a single static batch is the single most common reason a
pipeline that "worked in the demo" breaks the first time it runs against a second day of real
data.
{: .important }

## Volume and shape targets

Small enough to iterate on quickly, large enough that a broken join or an unindexed lookup is
visible before production: roughly 2,000 customers, 400 products across 6-8 categories, and
10,000-15,000 orders (with 1-4 line items each) in batch zero, with inventory snapshots,
clickstream events, and fulfillment records generated to match -- one inventory row per product
per warehouse, several clickstream events per order (browse, add-to-cart, purchase), and one
fulfillment record per shipped order.

## What Lecture 5 builds against this plan

This lecture is the design; Lecture 5 is the implementation -- a Faker-based Python script that
generates batch zero against these targets, writes it to the staging volume, and a loader notebook
that moves it into the landing zone's source-specific subfolders (and, for the CDC-shaped tables,
into the `dev.raw_cdc` schema Lecture 3 created) so Section 2 has real, imperfect data to build
against.

<!-- prevnext:start -->

---

| [&larr; Previous: Environment Setup - Repo, Catalog, Schema, and Volume]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/environment-setup-repo-catalog-schema-and-volume/' | relative_url }}) | [Next: Seed the Project - Data Generator and Loader Notebook &rarr;]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/seed-the-project-data-generator-and-loader-notebook/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

