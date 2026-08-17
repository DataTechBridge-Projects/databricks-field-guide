---
title: "Design the Silver Layer Data Quality Rules and Approach"
parent: "StepRight - Transformation Layers"
grand_parent: "StepRight Capstone Project"
nav_order: 1
permalink: /02-stepright-capstone-project/03-stepright-transformation-layers/design-the-silver-layer-data-quality-rules-and-approach/
read_minutes: 10
---

# Design the Silver Layer Data Quality Rules and Approach
{: .no_toc }

*Estimated read: 10 min*

Section 2's bronze layer answers one question: is this row structurally usable. Silver answers a
harder one: is this row *correct*, by StepRight's actual business rules. This lecture designs
silver as that data quality gate -- twelve rules across four categories, almost all of them
report-only, a deliberately small number of them serious enough to hold a row back entirely.

## Structural checks vs. business rules, again

Section 2, Lecture 5 drew a line between structural checks (bronze: is a primary key null, does a
foreign key resolve) and business rules (silver: is this value plausible, is this row internally
consistent). A row can pass every bronze structural check and still be wrong in a way only
business context reveals -- a `discount_pct` of 150% is a perfectly valid number, present and
non-null, that no reasonable StepRight discount should ever produce. That's silver's job to catch,
not bronze's.

## Twelve rules, four categories

| Category | Rule | Table | Severity |
|---|---|---|---|
| Completeness | Missing `region` | `customers` | Report-only |
| Completeness | Missing `order_date` | `orders` | Report-only |
| Completeness | Missing `quantity` | `order_items` | Report-only |
| Validity | `region` not a known value | `customers` | Report-only |
| Validity | `status` not a known value | `orders` | Report-only |
| Validity | `discount_pct` outside 0-100 | `order_items` | **Critical** |
| Referential consistency | `customer_id` doesn't resolve to a current customer | `orders` | **Critical** |
| Referential consistency | `product_id` doesn't resolve to a current product | `order_items` | **Critical** |
| Referential consistency | `order_id` doesn't resolve to a current order | `order_items` | Report-only |
| Business logic | Net price (`unit_price` after discount) negative | `order_items` | Report-only |
| Business logic | `order_date` in the future | `orders` | Report-only |
| Business logic | `quantity` over 10 | `order_items` | Report-only |

Nine of twelve are report-only; three are critical. That ratio is deliberate, not an accident of
which rules were easy to write.

## Why report-only is the default

A quality gate that quarantines on every rule violation stops being useful the moment it's too
aggressive -- analysts lose trust in a gold table that's missing rows for reasons nobody can
explain quickly, and engineers spend more time tuning thresholds than building anything new.
Report-only rules **tag** a row (an array of which rules it failed) but let it flow into silver
regardless -- visible for Section 7's monitoring dashboard, but never silently hidden from anyone
downstream. A future order dated tomorrow is worth flagging as unusual; it isn't a reason to drop
an otherwise-good order from gold-layer revenue reporting.
{: .important }

## Why these three are critical

Quarantine is reserved for rules where letting the row through actively corrupts something
downstream, not just looks unusual:

- **`discount_pct` outside 0-100** -- feeds directly into Section 4's revenue calculations
  (`gross`, `discount`, `net`). A 150% discount doesn't just look wrong; it produces a negative net
  revenue number that would silently skew every rollup built on top of it.
- **`customer_id` doesn't resolve** -- Section 4's customer 360 report and any customer-level
  aggregation has nothing to attach this order to. A resolvable customer isn't optional context;
  it's the join key gold depends on.
- **`product_id` doesn't resolve** -- same argument for product-level reporting: category
  breakdowns, sales velocity, and inventory joins all key off a product that has to exist.

Contrast this with the report-only `order_id` doesn't resolve rule for `order_items`: a line item
that arrives before its parent order (a CDC ordering quirk, not a data quality failure) usually
resolves itself the moment the next micro-batch processes the parent order -- worth flagging, not
worth quarantining a row that will almost certainly self-correct.

## Where quarantine happens relative to `AUTO CDC`

Critical-rule quarantine has to happen **before** the `AUTO CDC` merge, not after -- `AUTO CDC`
expects a clean stream of change events to merge into SCD Type 2 history, and a table already
holding an invalid row can't be "un-merged" the way a row can be excluded from a source stream.
Report-only tags, by contrast, ride through the merge and land in the final SCD2 table as an
ordinary column -- visible on every version of every row, not blocking anything. Lecture 2 plans
that ordering in detail; Lecture 3 builds it.

## What silver produces

By the end of this section, `silver_customers`, `silver_orders`, and `silver_order_items` hold
full SCD Type 2 history, tagged with report-only warnings, built only from rows that passed every
critical check. `silver_products` holds a deduplicated, validated current view of the catalog.
Section 4's revenue, customer 360, and product performance gold tables read from these four --
`inventory`, `clickstream`, and `fulfillment` don't get the same SCD/business-rule treatment in
this section, since none of them need merge-based history or StepRight-specific business rules
beyond what bronze's quarantine pattern already applies; Section 4 reads their `bronze_*_valid`
tables directly where it needs them.

<!-- prevnext:start -->

---

| [&larr; Previous: StepRight - Transformation Layers]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/' | relative_url }}) | [Next: Planning and Designing Silver Pipeline for CDC Sources &rarr;]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-cdc-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

