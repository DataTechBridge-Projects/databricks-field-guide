---
title: "Customer 360 for Marketing"
parent: "StepRight - Gold Layer Reporting and Analysis"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/customer-360-for-marketing/
read_minutes: 4
---

# Customer 360 for Marketing
{: .no_toc }

*Estimated read: 4 min*

Marketing doesn't ask "what did we sell yesterday" -- it asks "who is this customer." A **customer
360** view answers that in one row per customer: current profile attributes plus lifetime behavior,
built for segmentation campaigns and loyalty targeting rather than daily financial reporting.

## What goes into one row per customer

`gold_customer_360` blends two different shapes of input: `silver_customers`, filtered to each
customer's current SCD Type 2 version, contributes profile facts (`region`, `email`, `signup_date`);
`silver_orders` and `silver_order_items`, aggregated across a customer's entire order history,
contribute lifetime behavior (`lifetime_line_items`, `lifetime_net_revenue`, `first_order_date`,
`last_order_date`). Neither input alone answers marketing's question -- profile without behavior
can't segment for a win-back campaign, and behavior without profile can't target by region.

```python
# transformations/gold_reporting.py (addition)
from pyspark.sql.functions import col, sum as spark_sum, count, min as spark_min, max as spark_max

@dp.materialized_view(
    name="gold_customer_360",
    comment="One row per customer: current profile plus lifetime order behavior, for marketing"
)
def gold_customer_360():
    customers = spark.read.table("silver_customers").filter("__END_AT IS NULL")

    order_items = spark.read.table("silver_order_items").filter("__END_AT IS NULL")
    orders = (
        spark.read.table("silver_orders")
        .filter("__END_AT IS NULL")
        .filter("status != 'CANCELLED'")
    )
    priced_orders = (
        order_items
        .join(orders, "order_id")
        .withColumn("net_revenue",
                     col("unit_price") * col("quantity") * (1 - col("discount_pct") / 100))
    )

    behavior = (
        priced_orders
        .groupBy("customer_id")
        .agg(
            count("order_item_id").alias("lifetime_line_items"),
            spark_sum("net_revenue").alias("lifetime_net_revenue"),
            spark_min("order_date").alias("first_order_date"),
            spark_max("order_date").alias("last_order_date"),
        )
    )

    return customers.join(behavior, "customer_id", "left")
```

## Why the join is `left`, not inner

An inner join between `customers` and `behavior` would silently drop every customer who signed up
but never completed an order -- a segment marketing specifically wants to target for a first-purchase
campaign, not lose from the report entirely. The `left` join keeps every current customer row; a
customer with no order history simply gets `null` for `lifetime_net_revenue` and the three other
behavior columns, which downstream queries can `COALESCE` to zero rather than treating as a missing
row.

## Why `net_revenue` is computed inline here, not read from `gold_daily_revenue`

Lecture 1's `gold_daily_revenue` already computes net revenue -- but at the `day, category, region`
grain, not per customer, and it doesn't carry `customer_id` at all in its final output. Recomputing
`net_revenue` here from `silver_order_items` and `silver_orders` directly, using the same
`unit_price * quantity * (1 - discount_pct / 100)` formula, is a small amount of duplicated logic in
exchange for a materialized view that doesn't depend on another gold table's schema staying stable.
Each of this section's five gold tables reads only from silver and bronze, never from each other --
keeping the dependency graph flat and every gold table independently rebuildable.

## Common mistakes

- **Joining `behavior` before `customers` instead of after.** Starting the chain from
  `priced_orders` and joining `customers` onto it, rather than the reverse, quietly turns the query
  back into an inner join in practice -- a customer with zero orders never appears in
  `priced_orders` at all, so joining `customers` onto it as anything other than the left side loses
  the exact rows this lecture built the `left` join to keep.
- **Forgetting the `status != 'CANCELLED'` filter here too.** It's easy to remember this filter in
  Lecture 1 and forget it applies just as much to a lifetime-value calculation -- a customer whose
  only two orders were both cancelled should show `lifetime_net_revenue` as `null`, not as revenue
  StepRight never actually collected.

## Verifying the result

```sql
-- Exactly one row per customer_id
SELECT customer_id, COUNT(*) FROM dev.step_right.gold_customer_360
GROUP BY customer_id HAVING COUNT(*) > 1;
-- Should return zero rows

-- Customers with no order history yet -- a real marketing segment, not a bug
SELECT COUNT(*) FROM dev.step_right.gold_customer_360
WHERE lifetime_net_revenue IS NULL;

-- Highest lifetime value customers by region
SELECT region, customer_id, lifetime_net_revenue
FROM dev.step_right.gold_customer_360
ORDER BY lifetime_net_revenue DESC
LIMIT 20;
```

## What's next

`gold_customer_360` looks at a customer's whole history in one wide row. Lecture 3 turns the same
`silver_order_items` table toward a completely different grain -- not a customer, but a product, and
not just what sold, but what's still sitting on a warehouse shelf.

<!-- prevnext:start -->

---

| [&larr; Previous: Revenue Computation by Day, Category, Region]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/revenue-computation-by-day-category-region/' | relative_url }}) | [Next: Product Performance and Sales Velocity for Merchandising &rarr;]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/product-performance-and-sales-velocity-for-merchandising/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

