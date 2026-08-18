---
title: "Product Delivery and Fulfilment Health for Operations"
parent: "StepRight - Gold Layer Reporting and Analysis"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/product-delivery-and-fulfilment-health-for-operations/
read_minutes: 5
---

# Product Delivery and Fulfilment Health for Operations
{: .no_toc }

*Estimated read: 5 min*

Operations doesn't measure success in revenue -- it measures success in how long a shipped order
actually took to reach a customer, and how often that timeline breaks a promise StepRight made at
checkout. This section's last gold table closes the loop: `bronze_fulfillment_valid`, one record per
shipped order from the carrier feed, joined back against `silver_orders` to measure the two
intervals operations actually manages.

## Two intervals, not one

A single "days to deliver" number hides which half of the process is actually slow. This
materialized view keeps the two intervals separate on purpose: **time to ship** (`order_date` to
`ship_date` -- how long StepRight's own warehouse took to pack and hand off the order) and
**transit time** (`ship_date` to `delivery_date` -- how long the carrier took once it left the
building). A slow carrier and a slow warehouse look identical in a combined number and demand
completely different fixes.

```python
# transformations/gold_reporting.py (addition)
from pyspark.sql.functions import col, datediff, avg, sum as spark_sum, count, when, lit

SLA_DELIVERY_DAYS = 7

@dp.materialized_view(
    name="gold_fulfillment_health",
    comment="Time-to-ship and transit-time metrics by warehouse and carrier, for operations"
)
def gold_fulfillment_health():
    fulfillment = spark.read.table("bronze_fulfillment_valid")
    orders = (
        spark.read.table("silver_orders")
        .filter("__END_AT IS NULL")
        .filter("status = 'DELIVERED'")
    )

    joined = (
        fulfillment
        .join(orders, "order_id")
        .withColumn("days_to_ship", datediff(col("ship_date"), col("order_date")))
        .withColumn("transit_days", datediff(col("delivery_date"), col("ship_date")))
        .withColumn("total_days", datediff(col("delivery_date"), col("order_date")))
        .withColumn("sla_breach", col("total_days") > lit(SLA_DELIVERY_DAYS))
    )

    return (
        joined
        .groupBy("warehouse_id", "carrier", col("ship_date").alias("ship_date_day"))
        .agg(
            avg("days_to_ship").alias("avg_days_to_ship"),
            avg("transit_days").alias("avg_transit_days"),
            avg("total_days").alias("avg_total_days"),
            spark_sum(when(col("sla_breach"), 1).otherwise(0)).alias("sla_breaches"),
            count("order_id").alias("shipments"),
        )
    )
```

## Why this filters to `status = 'DELIVERED'`, not just current version

Every other gold table in this section either excludes `CANCELLED` orders or doesn't filter on
status at all. Fulfillment health needs the opposite discipline: `delivery_date` only means something
for an order that actually reached `DELIVERED`. An order still sitting at `SHIPPED` hasn't finished
its journey yet -- including it would compute a `transit_days` against a `delivery_date` that's
either `null` (correctly excluded by the join, since `bronze_fulfillment_valid` only gets a record
once a shipment event fires) or, worse, stale if the carrier feed hasn't caught up. Filtering to
`DELIVERED` up front keeps every row in this table a completed, measurable shipment.

## The SLA breach flag

`SLA_DELIVERY_DAYS = 7` turns a raw interval into the metric operations actually reports against: a
binary flag, aggregated into a breach count and an implied breach rate per warehouse and carrier.
Keeping the threshold as a module-level constant, rather than a magic number buried inside a `when`
clause, is what makes a future SLA renegotiation a one-line change instead of a search-and-replace
across the pipeline.

## Verifying the result

```sql
-- Worst-performing carrier by SLA breach rate, warehouses with meaningful volume only
SELECT
    carrier,
    SUM(shipments) AS total_shipments,
    SUM(sla_breaches) AS total_breaches,
    ROUND(SUM(sla_breaches) / SUM(shipments), 3) AS breach_rate
FROM dev.step_right.gold_fulfillment_health
GROUP BY carrier
HAVING SUM(shipments) >= 50
ORDER BY breach_rate DESC;

-- Warehouses whose own pack-and-ship time, not the carrier, is the bottleneck
SELECT warehouse_id, AVG(avg_days_to_ship) AS avg_days_to_ship
FROM dev.step_right.gold_fulfillment_health
GROUP BY warehouse_id
ORDER BY avg_days_to_ship DESC;
```

The first query isolates carrier performance; the second isolates warehouse performance -- exactly
the split `days_to_ship` vs. `transit_days` was built to enable, and exactly why a single combined
"delivery time" metric wouldn't have told operations which team to talk to.

## Common mistakes

- **Computing `total_days` as `days_to_ship + transit_days` instead of `datediff` against the
  original dates.** The two intervals should sum to the same value as a direct
  `order_date`-to-`delivery_date` `datediff`, but computing it independently, directly from the
  source dates, avoids silently compounding a rounding or null-handling bug from one interval into
  the total.
- **Filtering `status = 'DELIVERED'` after the join instead of before it.** Joining the full,
  unfiltered `silver_orders` against `fulfillment` first and filtering afterward still computes
  `datediff` against non-`DELIVERED` rows before they're dropped -- harmless here since the result is
  discarded, but wasted computation on a table this size, and a habit that causes real bugs the
  moment a lecture's filter needs to happen before an aggregation rather than after one.
{: .important }

## What Section 4 leaves ready for Section 5

Five gold materialized views, each reading only from silver and bronze, none of them dependent on
another gold table's schema: daily revenue for finance, a customer 360 for marketing, product
performance and stock risk for merchandising, a clickstream funnel for growth, and fulfillment
health for operations. Section 5 wires ingestion, transformation, and this reporting layer together
into one scheduled Lakeflow Job -- so all five of these tables refresh on a real production cadence,
not by someone remembering to click "run" every morning.

<!-- prevnext:start -->

---

| [&larr; Previous: Clickstream Funnel Analysis for Growth Teams]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/clickstream-funnel-analysis-for-growth-teams/' | relative_url }}) | [Next: StepRight - Orchestration and Job Scheduling &rarr;]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

