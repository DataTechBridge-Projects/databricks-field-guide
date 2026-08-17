---
title: "Designing the Gold Layer"
parent: "Medallion Architecture - Design and Implementation"
grand_parent: "Databricks Fundamentals"
nav_order: 4
permalink: /01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-gold-layer/
read_minutes: 6
---

# Designing the Gold Layer
{: .no_toc }

*Estimated read: 6 min*

Gold is the layer your stakeholders actually see -- built for a specific consumer's question, not
a general-purpose "clean copy" of anything. This lecture works through one concrete example end to
end: daily revenue by customer tier.

## Gold is consumer-shaped, not source-shaped

Where bronze mirrors the source and silver conforms to one consistent shape, gold is deliberately
**denormalized and pre-aggregated for a specific reporting need**. Multiple gold tables commonly
exist for different consumers, all built from the same silver tables:

```sql
CREATE TABLE gold.daily_revenue_by_tier
USING DELTA
AS
SELECT
    order_date,
    c.customer_tier,
    sum(o.order_total)                              AS gross_revenue,
    sum(o.order_total * (1 - o.discount_pct))        AS net_revenue,
    count(*)                                         AS order_count
FROM silver.orders o
JOIN silver.customers c ON o.customer_id = c.customer_id AND c.is_current = true
WHERE o.order_status = 'completed'
GROUP BY order_date, c.customer_tier;
```

## Materialized table vs. view: the choice that matters most

The description that seeded this lecture asks specifically why to choose a **materialized table**
over a **view** here -- worth answering directly:

| | View | Materialized table |
|---|---|---|
| Computed | On every query, from underlying tables | Once, on a schedule; stored as its own table |
| Query latency | Depends on underlying tables' full computation cost, every time | Fast -- just reading pre-computed rows |
| Freshness | Always current with underlying data | As current as the last refresh |
| Cost | Recomputed repeatedly, once per query | Computed once per refresh, reused by many queries |

**Key term:** for a dashboard queried repeatedly by many stakeholders throughout the day, a
**materialized table** refreshed once (e.g. nightly, or incrementally via Lakeflow Declarative
Pipelines in Section 9) is almost always the right choice -- recomputing the same aggregation from
scratch on every dashboard refresh wastes compute and adds latency for every viewer, for a result
that hasn't actually changed since the last data load anyway.
{: .important }

Views remain the right choice for genuinely ad hoc, exploratory queries where freshness matters
more than repeated-query cost, or where the underlying computation is cheap enough that
materializing it wouldn't meaningfully help.

## Designing a gold table: the questions to ask first

1. **Who is the consumer, specifically?** "Finance" and "marketing" often need genuinely different
   shapes of the same underlying facts -- don't build one generic gold table trying to serve both.
2. **What grain does the consumer actually need?** Daily, not hourly, if that's all the dashboard
   shows -- a finer grain than needed just adds unused storage and compute cost.
3. **Does this need to be a table or a view?** Repeated, latency-sensitive access -> table.
   Infrequent, exploratory, or always-must-be-live access -> view.

<!-- prevnext:start -->

---

| [&larr; Previous: Designing the Silver Layer]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-silver-layer/' | relative_url }}) | [Next: Architecture Review and Design Decisions &rarr;]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/architecture-review-and-design-decisions/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
