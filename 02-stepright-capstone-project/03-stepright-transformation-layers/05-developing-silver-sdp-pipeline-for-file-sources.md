---
title: "Developing Silver SDP Pipeline for File Sources"
parent: "StepRight - Transformation Layers"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/03-stepright-transformation-layers/developing-silver-sdp-pipeline-for-file-sources/
read_minutes: 6
---

# Developing Silver SDP Pipeline for File Sources
{: .no_toc }

*Estimated read: 6 min*

Lecture 4 planned latest-wins deduplication and light validity tagging for `silver_products`. This
lecture builds it -- deliberately as a batch table, not a streaming one, since deduplicating
"latest row per key" needs to see the whole table at once.

## Why this table reads in batch, not as a stream

Every silver table built in Lecture 3 used `spark.readStream` -- appropriate for incrementally
merging CDC changes. Picking the single newest row per `product_id` is a different kind of
computation: it needs a window function ranking every version of a product against every other
version, which requires visibility into the full table, not just a stream's newest micro-batch.
Because this pipeline runs in **triggered** mode (Section 2's `continuous: false`), a full batch
recompute on every run is both correct and inexpensive at StepRight's data volumes -- `silver_products`
is defined with a plain `spark.read`, not `spark.readStream`.

## The dedup and tagging logic

```python
# transformations/silver_files.py
import dlt as dp
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col
from dq_helpers import tag_business_rules

KNOWN_CATEGORIES = ["Running", "Trail", "Basketball", "Casual", "Training", "Sandals"]

@dp.table(
    name="silver_products",
    comment="Deduplicated, validated current view of the product catalog"
)
def silver_products():
    latest_per_product = Window.partitionBy("product_id").orderBy(col("_ingested_at").desc())

    deduped = (
        spark.read.table("dev.step_right.bronze_products_valid")
        .withColumn("_rn", row_number().over(latest_per_product))
        .filter(col("_rn") == 1)
        .drop("_rn")
    )

    warnings = {
        "invalid_list_price": col("list_price") <= 0,
        "unknown_category": ~col("category").isin(KNOWN_CATEGORIES),
    }
    return tag_business_rules(deduped, critical_checks={}, warning_checks=warnings)
```

`row_number().over(latest_per_product)` ranks every version of a given `product_id` by
`_ingested_at`, newest first; filtering to `_rn == 1` keeps exactly one row per product -- the same
"latest wins" logic a legacy warehouse's SCD Type 1 dimension load would apply, just expressed as a
window function instead of a `MERGE ... WHEN MATCHED` statement. `tag_business_rules` is the same
helper Lecture 3 introduced, called here with an empty `critical_checks` dictionary for the same
reason `silver_customers` used one -- these two validity checks are worth flagging, not worth
holding a product back from the catalog over.

## Verifying the result

```sql
-- Exactly one row per product_id
SELECT product_id, COUNT(*) AS versions
FROM dev.step_right.silver_products
GROUP BY product_id
HAVING COUNT(*) > 1;
-- Should return zero rows

-- Report-only warnings on the current catalog
SELECT explode(_dq_warnings) AS warning, COUNT(*) AS product_count
FROM dev.step_right.silver_products
GROUP BY warning;
```

The first query is the real correctness check for this table -- any `product_id` with more than
one row means the dedup logic missed something, most likely two rows sharing the exact same
`_ingested_at` timestamp (batch-loaded files landing in the same micro-batch), which `row_number()`
breaks ties on arbitrarily. If that happens in practice, add a secondary `orderBy` column -- file
name or an explicit version number from the source -- rather than relying on timestamp precision
alone.

## Common mistakes

- **Using `spark.readStream` out of habit.** `Window.partitionBy(...).orderBy(...)` without a
  watermark isn't supported against an unbounded streaming DataFrame -- this table has to read in
  batch, which is the correct choice here, not a workaround.
- **Forgetting this table needs to fully recompute, not just append.** Because `silver_products` is
  a batch table over the *entire* `bronze_products_valid` history each run, it should be defined as
  a standard `@dp.table`, not registered as an append-flow target -- SDP handles the difference
  automatically based on how the table is declared, but it's worth understanding why this table's
  definition looks different from Lecture 3's `AUTO CDC` targets.

## What Section 3 leaves ready for Section 4

Four silver tables -- three with full SCD Type 2 history via `AUTO CDC`, one deduplicated and
validated in batch -- plus three raw-but-quarantine-clean bronze tables (`inventory`,
`clickstream`, `fulfillment`) that never needed a silver layer of their own. Section 4 builds five
gold tables on top of exactly this foundation, starting with the revenue report that
`silver_orders` and `silver_order_items` were built to support.

<!-- prevnext:start -->

---

| [&larr; Previous: Planning and Designing Silver Pipeline for File Sources]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-file-sources/' | relative_url }}) | [Next: StepRight - Gold Layer Reporting and Analysis &rarr;]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

