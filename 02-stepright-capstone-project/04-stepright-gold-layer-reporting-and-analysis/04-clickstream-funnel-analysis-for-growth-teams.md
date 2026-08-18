---
title: "Clickstream Funnel Analysis for Growth Teams"
parent: "StepRight - Gold Layer Reporting and Analysis"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/clickstream-funnel-analysis-for-growth-teams/
read_minutes: 4
---

# Clickstream Funnel Analysis for Growth Teams
{: .no_toc }

*Estimated read: 4 min*

Growth doesn't care about completed orders alone -- it cares about everyone who *didn't* finish
checking out. `bronze_clickstream_valid` is the only input Section 1's plan gave three event types
for: `browse`, `add_to_cart`, and `purchase`, one row per event, generated several times over per
order the way a real storefront actually behaves. This lecture turns that raw event stream into a
funnel.

## Why this reads bronze, not silver, again

Clickstream is the second table in this section -- after inventory -- that skips silver entirely,
for the same reason Section 3's design note gave: no StepRight-specific business rule beyond
bronze's structural quarantine applies to a page-view event. `bronze_clickstream_valid` is already
the clean input this materialized view needs.

## Counting each stage of the funnel

```python
# transformations/gold_reporting.py (addition)
from pyspark.sql.functions import col, countDistinct, when

@dp.materialized_view(
    name="gold_clickstream_funnel",
    comment="Daily browse -> add-to-cart -> purchase funnel by product category, for growth"
)
def gold_clickstream_funnel():
    clickstream = spark.read.table("bronze_clickstream_valid")
    products = spark.read.table("silver_products")

    events = clickstream.join(products, "product_id", "left")

    return (
        events
        .groupBy(col("event_timestamp").cast("date").alias("event_date"), "category")
        .agg(
            countDistinct(when(col("event_type") == "browse", col("session_id"))).alias("browse_sessions"),
            countDistinct(when(col("event_type") == "add_to_cart", col("session_id"))).alias("cart_sessions"),
            countDistinct(when(col("event_type") == "purchase", col("session_id"))).alias("purchase_sessions"),
        )
    )
```

`countDistinct` on `session_id`, gated by a `when` per stage, is what turns three event types into
three funnel counts in one pass over the same joined DataFrame -- no need to filter and re-aggregate
the same table three separate times. Counting **sessions**, not raw events, matters here: a single
browsing session that views the same product five times should count once toward `browse_sessions`,
not five times, or the funnel overstates top-of-funnel traffic relative to the stages below it.

## Conversion rates as a derived view, not a stored column

`gold_clickstream_funnel` deliberately stops at raw counts rather than storing a computed
`conversion_rate` column -- a ratio derived from two other columns already in the table is cheap to
compute at query time and never risks drifting out of sync with the counts it's based on:

```sql
SELECT
    event_date,
    category,
    browse_sessions,
    cart_sessions,
    purchase_sessions,
    ROUND(cart_sessions / NULLIF(browse_sessions, 0), 3) AS browse_to_cart_rate,
    ROUND(purchase_sessions / NULLIF(cart_sessions, 0), 3) AS cart_to_purchase_rate
FROM dev.step_right.gold_clickstream_funnel
ORDER BY event_date DESC;
```

`NULLIF(browse_sessions, 0)` guards the same division-by-zero case Lecture 3's `days_of_supply`
handled with a `when` -- a category with zero browse sessions on a given day should produce a `null`
rate, not an error or a misleading zero.

## Where the funnel narrows, and why the left join matters

`events` joins `clickstream` to `products` as `left`, not inner -- a `purchase` event's `product_id`
should always resolve, since it came from an order that also passed silver's referential checks, but
an early `browse` event can reference a product a shopper viewed and StepRight later delisted. Losing
that row to an inner join would understate `browse_sessions` for a category that's otherwise
performing well; keeping it with a `null` category groups it separately instead, visible rather than
silently dropped.

## Verifying the result

```sql
-- Categories with a sharp cart-to-purchase drop -- growth's highest-priority investigation list
SELECT category, SUM(cart_sessions) AS carts, SUM(purchase_sessions) AS purchases
FROM dev.step_right.gold_clickstream_funnel
GROUP BY category
HAVING SUM(cart_sessions) > 0
ORDER BY (SUM(purchase_sessions) / SUM(cart_sessions)) ASC;
```

## What's next

The funnel tracks a shopper up to the purchase. Lecture 5 picks up exactly where this one leaves
off -- what happens to a product after the purchase event fires.

<!-- prevnext:start -->

---

| [&larr; Previous: Product Performance and Sales Velocity for Merchandising]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/product-performance-and-sales-velocity-for-merchandising/' | relative_url }}) | [Next: Product Delivery and Fulfilment Health for Operations &rarr;]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/product-delivery-and-fulfilment-health-for-operations/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

