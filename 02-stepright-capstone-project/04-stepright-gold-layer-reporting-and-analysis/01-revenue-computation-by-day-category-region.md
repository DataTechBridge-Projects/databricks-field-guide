---
title: "Revenue Computation by Day, Category, Region"
parent: "StepRight - Gold Layer Reporting and Analysis"
grand_parent: "StepRight Capstone Project"
nav_order: 1
permalink: /02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/revenue-computation-by-day-category-region/
read_minutes: 10
---

# Revenue Computation by Day, Category, Region
{: .no_toc }

*Estimated read: 10 min*

Four silver tables, three of them full SCD Type 2 history. This lecture builds the gold table
finance actually asks for -- one row per day, category, and region, with **gross revenue**,
**discount**, and **net revenue** -- and along the way surfaces the two mistakes that quietly wreck
a revenue number built on top of Type 2 history: forgetting to filter to the current version, and
forgetting that a cancelled order isn't revenue.

## What finance actually wants

Nobody downstream wants `silver_order_items` -- it's one row per line item, keyed for correctness,
not for a dashboard. Finance wants a daily rollup they can slice by product category and customer
region without writing a four-table join every time: `gross_revenue`, `discount_amount`, and
`net_revenue`, grouped by `revenue_date`, `category`, and `region`. Building that means pulling in
all four silver tables Section 3 produced:

| Silver table | What it contributes |
|---|---|
| `silver_order_items` | `unit_price`, `quantity`, `discount_pct` -- the actual dollars |
| `silver_orders` | `order_date`, `status`, and the join key back to `customer_id` |
| `silver_customers` | `region` -- the dimension finance wants to slice by |
| `silver_products` | `category` -- the other dimension finance wants to slice by |

## Gross, discount, and net -- and what "proportional allocation" means here

`discount_pct` already lives on `silver_order_items`, at the line-item grain -- which means the
dollar discount for a given line naturally allocates **in proportion to that line's own gross
revenue**, not as some order-level dollar figure split evenly across items. A $150 running shoe and
a $12 pair of socks on the same order, both discounted 20%, each carry a discount proportional to
their own price -- $30 and $2.40 -- rather than splitting a single order-level discount amount down
the middle and distorting both lines. That's the calculation, expressed directly:

```python
gross_revenue = unit_price * quantity
discount_amount = gross_revenue * (discount_pct / 100)
net_revenue = gross_revenue - discount_amount
```

Contrast this with a legacy warehouse pattern that's easy to reach for out of habit: storing one
discount dollar amount at the order header and dividing it evenly across line items regardless of
each item's price. That approach was often a compromise forced by an OLTP schema that didn't carry
a per-line discount column at all. StepRight's line-item-level `discount_pct` means the correct,
proportional allocation isn't extra work here -- it's just multiplication.

## Worked example with real numbers

Three line items on the same order, running shoes at $150 and socks at $12, both carrying a 20%
`discount_pct`, plus a pair of sandals at $60 with no discount at all:

| `product_id` | `unit_price` | `quantity` | `discount_pct` | `gross_revenue` | `discount_amount` | `net_revenue` |
|---|---|---|---|---|---|---|
| PROD-00042 (shoes) | 150.00 | 1 | 20 | 150.00 | 30.00 | 120.00 |
| PROD-00187 (socks) | 12.00 | 2 | 20 | 24.00 | 4.80 | 19.20 |
| PROD-00301 (sandals) | 60.00 | 1 | 0 | 60.00 | 0.00 | 60.00 |

Summed into `gold_daily_revenue` for that order's day, category, and region: `gross_revenue` of
234.00, `discount_amount` of 34.80, `net_revenue` of 199.20. Every line kept its own discount dollar
figure, proportional to its own gross -- the sandals line, with a 0% `discount_pct`, contributes
nothing to `discount_amount` at all, exactly as it should. A flat order-level discount split evenly
three ways would have shaved roughly $11.60 off each line regardless of price, understating the
socks line's *effective* discount rate and overstating the shoes line's -- a distortion that
compounds the moment someone tries to explain "average discount rate by category" from a number that
was never really allocated by category to begin with.

## Building `gold_daily_revenue`

```python
# transformations/gold_reporting.py
import dlt as dp
from pyspark.sql.functions import col, sum as spark_sum, count

@dp.materialized_view(
    name="gold_daily_revenue",
    comment="Daily gross, discount, and net revenue by product category and customer region"
)
def gold_daily_revenue():
    order_items = spark.read.table("silver_order_items").filter("__END_AT IS NULL")
    orders = (
        spark.read.table("silver_orders")
        .filter("__END_AT IS NULL")
        .filter("status != 'CANCELLED'")
    )
    customers = spark.read.table("silver_customers").filter("__END_AT IS NULL")
    products = spark.read.table("silver_products")

    priced = (
        order_items
        .join(orders, "order_id")
        .join(customers, "customer_id")
        .join(products, "product_id")
        .withColumn("gross_revenue", col("unit_price") * col("quantity"))
        .withColumn("discount_amount", col("gross_revenue") * (col("discount_pct") / 100))
        .withColumn("net_revenue", col("gross_revenue") - col("discount_amount"))
    )

    return (
        priced
        .groupBy(col("order_date").alias("revenue_date"), col("category"), col("region"))
        .agg(
            spark_sum("gross_revenue").alias("gross_revenue"),
            spark_sum("discount_amount").alias("discount_amount"),
            spark_sum("net_revenue").alias("net_revenue"),
            count("order_item_id").alias("line_item_count"),
        )
    )
```

Two filters on `orders` do real work before a single dollar gets summed: `__END_AT IS NULL` and
`status != 'CANCELLED'`.

## The same logic, if you'd rather declare it in SQL

Lakeflow Declarative Pipelines lets a materialized view be declared as SQL instead of PySpark --
worth knowing if you're coming from a legacy warehouse background where every transformation was a
stored procedure or a scheduled SQL script, not a Python DataFrame chain. The two are equivalent, not
a fallback for a "simpler" case:

```sql
-- transformations/gold_daily_revenue.sql
CREATE MATERIALIZED VIEW gold_daily_revenue
COMMENT "Daily gross, discount, and net revenue by product category and customer region"
AS
SELECT
    o.order_date AS revenue_date,
    p.category,
    c.region,
    SUM(oi.unit_price * oi.quantity) AS gross_revenue,
    SUM(oi.unit_price * oi.quantity * (oi.discount_pct / 100)) AS discount_amount,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS net_revenue,
    COUNT(oi.order_item_id) AS line_item_count
FROM silver_order_items oi
JOIN silver_orders o ON oi.order_id = o.order_id AND o.__END_AT IS NULL AND o.status != 'CANCELLED'
JOIN silver_customers c ON o.customer_id = c.customer_id AND c.__END_AT IS NULL
JOIN silver_products p ON oi.product_id = p.product_id
WHERE oi.__END_AT IS NULL
GROUP BY o.order_date, p.category, c.region;
```

Every filter that mattered in the PySpark version -- both `__END_AT IS NULL` checks, `status !=
'CANCELLED'` -- has a direct SQL equivalent, folded into the `JOIN ... ON` and `WHERE` clauses rather
than a chain of `.filter()` calls. Which form a given pipeline uses is a team preference, not a
capability difference; a project can mix SQL and Python materialized views in the same
`transformations/` folder, the same way `pipeline.yml`'s `glob: include` pattern already picks up
both `.py` and `.sql` files without extra configuration.

## Why `__END_AT IS NULL` isn't optional

`silver_order_items`, `silver_orders`, and `silver_customers` all hold **full SCD Type 2 history**
-- every prior version of every row, each with its own `__START_AT`/`__END_AT` window. Reading one
of these tables without a current-version filter doesn't just risk stale data; it silently
**multiplies revenue** by the number of historical versions a row happens to have. An order that
changed status from `PLACED` to `SHIPPED` to `DELIVERED` has three rows in `silver_orders` -- join
that against `silver_order_items` without `__END_AT IS NULL` on both sides, and that order's line
items get summed three times. `silver_products` is the one exception in this join: Section 3's
Lecture 5 built it as a plain deduplicated table, one row per `product_id`, so it needs no
current-version filter at all -- there's no history to filter out.

## Why `status != 'CANCELLED'` matters just as much

A cancelled order still has line items sitting in `silver_order_items` with real `unit_price` and
`quantity` values -- nothing about cancellation deletes or zeroes them out, because Section 3's
design deliberately preserves full history rather than mutating rows. Left unfiltered, a spike in
cancellations shows up as a corresponding spike in reported revenue, which is exactly backwards from
what actually happened. This is a genuine business decision, not a technical default -- StepRight
could equally well build a *second* gold table tracking cancelled-order value for a different
audience (loss analysis, fraud signals), but `gold_daily_revenue` itself represents revenue finance
can act on, which cancelled orders are not.

## Refresh strategy: why this one is a full recompute

[Part 1, Section 9]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-gold-layer-materialized-views-and-incremental-refresh/' | relative_url }})
drew the line between **full** and **incremental** materialized view refresh: simple, additive
aggregations over append-only sources are the best candidates for incremental refresh; complex joins
and non-append sources tend to force a full recompute. `gold_daily_revenue` sits firmly in the
second bucket -- a four-way join against three SCD Type 2 tables, each of which can rewrite a row's
`__END_AT` in place rather than only appending, gives SDP's refresh planner little room to determine
which output groups a given upstream change could possibly affect. In practice that means every
pipeline run recomputes this materialized view from scratch, which is the correct, expected trade
for correctness here -- StepRight's order volume is small enough that a full recompute costs seconds,
not minutes, and a wrong incremental update that missed a status change would be far more expensive
to debug than the extra compute.

## Clustering `gold_daily_revenue` for how it's actually queried

Every query in the previous section filters or groups by `revenue_date`, `category`, or `region` --
exactly the columns worth **Liquid Clustering** on, so Databricks can skip files that don't match a
given filter instead of scanning the whole table:

```sql
ALTER TABLE dev.step_right.gold_daily_revenue
CLUSTER BY (revenue_date, category, region);
```

This is the same clustering discipline [Part 1, Section 5]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/' | relative_url }})
introduced for silver and bronze tables, applied here to the table BI dashboards actually hit --
worth doing on a gold table even more than on the layers beneath it, since a gold table is where
query concurrency and dashboard refresh latency are felt directly by the people StepRight built this
whole pipeline for.

## Verifying the numbers

```sql
-- Sanity check: does net revenue reconcile against gross minus discount?
SELECT * FROM dev.step_right.gold_daily_revenue
WHERE ABS(net_revenue - (gross_revenue - discount_amount)) > 0.01;
-- Should return zero rows

-- Top category by net revenue, last 30 days
SELECT category, SUM(net_revenue) AS total_net_revenue
FROM dev.step_right.gold_daily_revenue
WHERE revenue_date >= date_sub(current_date(), 30)
GROUP BY category
ORDER BY total_net_revenue DESC;

-- Revenue by region, sanity-checked against known region values
SELECT region, SUM(net_revenue) AS total_net_revenue, SUM(line_item_count) AS items_sold
FROM dev.step_right.gold_daily_revenue
GROUP BY region
ORDER BY total_net_revenue DESC;
```

The first query is a structural check that should always return zero rows -- if it doesn't, the
aggregation logic itself is wrong, not the underlying data. The other two are the kind of query
finance actually runs against this table daily.

## Common mistakes

- **Filtering `__END_AT IS NULL` on only one SCD2 table.** A join between an unfiltered
  `silver_orders` and a filtered `silver_order_items` still fans out on the unfiltered side -- every
  table carrying history needs its own filter, not just one of them.
- **Applying `status != 'CANCELLED'` after the `groupBy` instead of before it.** Filtering the
  aggregated `gold_daily_revenue` output for a specific status doesn't remove cancelled orders from
  the sums that already happened -- the filter has to sit on `orders` before the join, the same
  lesson Section 3 taught about filtering `silver_orders_clean` before `AUTO CDC`'s merge rather
  than after.
- **Reaching for `spark.readStream` here out of habit.** This materialized view reads all four
  inputs in batch (`spark.read`), the same reasoning Section 3, Lecture 5 used for `silver_products`
  -- a `groupBy` aggregation across arbitrary historical rows isn't expressible as a simple streaming
  append, and SDP's materialized view refresh handles recomputation efficiently on its own without
  the pipeline author managing streaming state by hand.
{: .important }

## What's next

`gold_daily_revenue` answers finance's question. Lecture 2 turns the same silver layer toward a
completely different consumer -- marketing wants to know who each customer *is*, not just what they
bought on a given day.

<!-- prevnext:start -->

---

| [&larr; Previous: StepRight - Gold Layer Reporting and Analysis]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/' | relative_url }}) | [Next: Customer 360 for Marketing &rarr;]({{ '/02-stepright-capstone-project/04-stepright-gold-layer-reporting-and-analysis/customer-360-for-marketing/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

