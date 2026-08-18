---
title: "Product Performance and Sales Velocity for Merchandising"
parent: "StepRight - Gold Layer Reporting and Analysis"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/product-performance-and-sales-velocity-for-merchandising/
read_minutes: 5
---

# Product Performance and Sales Velocity for Merchandising
{: .no_toc }

*Estimated read: 5 min*

Merchandising asks a question neither of the last two gold tables answer: for a given product, how
fast is it selling, and is there enough stock on hand to keep selling it. That second half needs
`bronze_inventory_valid` -- the first table in this section to read from bronze directly instead of
silver, and the first place gold has to do its own deduplication work.

## Units sold and revenue, from `silver_order_items` alone

The sales half is a straightforward aggregation, the same shape as Lecture 1's revenue rollup but
grouped by `product_id` instead of day/category/region:

```python
# transformations/gold_reporting.py (addition)
from pyspark.sql.functions import col, sum as spark_sum, row_number, when, lit
from pyspark.sql.window import Window

@dp.materialized_view(
    name="gold_product_performance",
    comment="Units sold, revenue, current stock, and stock risk per product, for merchandising"
)
def gold_product_performance():
    order_items = spark.read.table("silver_order_items").filter("__END_AT IS NULL")
    products = spark.read.table("silver_products")

    sales = (
        order_items
        .withColumn("net_revenue",
                     col("unit_price") * col("quantity") * (1 - col("discount_pct") / 100))
        .groupBy("product_id")
        .agg(
            spark_sum("quantity").alias("units_sold"),
            spark_sum("net_revenue").alias("net_revenue"),
        )
    )
```

Nothing new here -- `__END_AT IS NULL` on the SCD2 side, a `groupBy` on the join key merchandising
cares about. The interesting part is what comes next.

## Why inventory needs its own dedup, right here in gold

Section 3's design note flagged this directly: `inventory`, `clickstream`, and `fulfillment` "don't
get the same SCD/business-rule treatment... Section 4 reads their `bronze_*_valid` tables directly."
That's a real consequence, not a footnote -- `bronze_inventory_valid` is an Auto Loader table that
appends a new snapshot row every time the 3PL drops a fresh inventory file, and nothing upstream of
this materialized view has ever deduplicated it down to "the current quantity per product per
warehouse." Unlike `silver_products`, which Section 3, Lecture 5 already reduced to one row per
product with a window function, inventory arrives at gold still holding every historical snapshot.
The same "latest wins" pattern applies here, just one layer later than it did for products:

```python
    latest_snapshot = Window.partitionBy("product_id", "warehouse_id").orderBy(col("_ingested_at").desc())

    current_inventory = (
        spark.read.table("bronze_inventory_valid")
        .withColumn("_rn", row_number().over(latest_snapshot))
        .filter(col("_rn") == 1)
        .drop("_rn")
        .groupBy("product_id")
        .agg(spark_sum("quantity_on_hand").alias("current_stock"))
    )
```

`row_number()` ranks every snapshot for a given `product_id, warehouse_id` pair by `_ingested_at`,
newest first, then keeps only the top-ranked row per warehouse -- the same technique
`silver_products` used, deliberately not moved into a silver table of its own, since a materialized
view recomputes this on every refresh at negligible cost for StepRight's data volume. The `groupBy`
that follows sums stock across every warehouse a product sits in, since merchandising's question is
"how much do we have," not "how much is in warehouse 3" specifically.

## Sales velocity and stock risk

With `sales` and `current_inventory` both keyed on `product_id`, the join produces the metric
merchandising actually acts on -- not just current stock, but whether that stock will last:

```python
    return (
        sales
        .join(products, "product_id")
        .join(current_inventory, "product_id", "left")
        .withColumn("units_per_day", col("units_sold") / lit(30))
        .withColumn(
            "days_of_supply",
            when(col("units_per_day") > 0, col("current_stock") / col("units_per_day"))
            .otherwise(lit(None)),
        )
        .withColumn("stock_risk", col("days_of_supply") < lit(14))
    )
```

`units_per_day` treats `units_sold` as a trailing-30-day figure (the window `silver_order_items`
itself covers, per Section 1's batch-zero generation targets); `days_of_supply` divides current
stock by that daily rate to answer "at this pace, how many days until this product sells out";
`stock_risk` flags anything under a two-week runway. A product with `current_stock` of `null` --
one with sales but no inventory snapshot yet -- correctly produces a `null` `days_of_supply` rather
than a division error or a false zero, which is exactly why `current_inventory` joins as `left`
rather than inner.

## Why 30 days, not the whole table's history

`units_per_day` divides `units_sold` by a flat 30 -- an approximation that's only honest if
`units_sold` itself is scoped to a comparable window. As written here, `sales` aggregates every
order line item StepRight has ever recorded, which understates velocity for a product that sold
briskly in its first month and then went quiet, or overstates it for a product that just launched.
A production version of this table would add a `.filter(col("order_date") >= date_sub(current_date(),
30))` on `order_items` before the `groupBy`, scoping `units_sold` to a genuine trailing 30-day
window -- left out of the version above to keep the join logic the focus, but worth building before
trusting `stock_risk` for a real reorder decision.

## Verifying the result

```sql
-- Products at stock risk, worst runway first
SELECT product_id, category, units_sold, current_stock, days_of_supply
FROM dev.step_right.gold_product_performance
WHERE stock_risk = true
ORDER BY days_of_supply ASC;

-- Best sellers with no inventory snapshot yet -- a gap to flag, not silently ignore
SELECT product_id, units_sold, net_revenue
FROM dev.step_right.gold_product_performance
WHERE current_stock IS NULL
ORDER BY units_sold DESC;
```

## Common mistakes

- **Deduplicating inventory per `product_id` alone, skipping `warehouse_id`.** Partitioning the
  window only by `product_id` picks one warehouse's snapshot and silently discards every other
  warehouse's stock for that product -- the partition key has to match the actual grain of a
  snapshot row.
- **Joining `current_inventory` as inner instead of left.** A product that sold well in the trailing
  window but hasn't had an inventory file land yet is exactly the row merchandising most needs to
  see -- an inner join makes it disappear instead of surfacing it with a `null` stock figure.
{: .important }

## What's next

Sales velocity looks backward at completed orders. Lecture 4 looks one step earlier in the customer
journey -- not what StepRight sold, but what shoppers did on the site before they bought anything at
all.

<!-- prevnext:start -->

---

| [&larr; Previous: Customer 360 for Marketing]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/customer-360-for-marketing/' | relative_url }}) | [Next: Clickstream Funnel Analysis for Growth Teams &rarr;]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/clickstream-funnel-analysis-for-growth-teams/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

