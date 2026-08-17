---
title: "Developing Silver Pipeline for AUTO CDC Sources"
parent: "StepRight - Transformation Layers"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/03-stepright-transformation-layers/developing-silver-pipeline-for-auto-cdc-sources/
read_minutes: 15
---

# Developing Silver Pipeline for AUTO CDC Sources
{: .no_toc }

*Estimated read: 15 min*

Lecture 2 planned tag, split, then merge for all three CDC-sourced tables. This lecture builds it
-- a two-tier tagging helper, one fully worked table with a referential join, and the `AUTO CDC`
flows that turn a clean stream into full SCD Type 2 history.

## Extending Section 2's tagging helper for two severities

Bronze's `tag_quality` produced one pass/fail flag. Silver needs two tiers -- critical and
warning -- so a new helper builds both from two separate rule dictionaries:

```python
# transformations/dq_helpers.py (addition)
from pyspark.sql.functions import when, lit, array, array_except, size, col

def tag_business_rules(df, critical_checks: dict, warning_checks: dict):
    critical_flags = array(*[when(cond, lit(name)) for name, cond in critical_checks.items()])
    warning_flags = array(*[when(cond, lit(name)) for name, cond in warning_checks.items()])
    return (
        df
        .withColumn("_dq_critical_failures",
                    array_except(critical_flags, array(lit(None).cast("string"))))
        .withColumn("_dq_critical_failed", size(col("_dq_critical_failures")) > 0)
        .withColumn("_dq_warnings",
                    array_except(warning_flags, array(lit(None).cast("string"))))
    )
```

An empty `critical_checks` dictionary (customers, below) is a valid input -- `_dq_critical_failed`
just comes back `false` for every row, and every row proceeds to the merge.

## Worked example: `silver_orders`

```python
# transformations/silver_cdc.py
import dlt as dp
from pyspark.sql.functions import col, current_date
from dq_helpers import tag_business_rules

KNOWN_STATUSES = ["PLACED", "SHIPPED", "DELIVERED", "CANCELLED"]

@dp.table(name="silver_orders_tagged", comment="orders tagged with business rule outcomes")
def silver_orders_tagged():
    orders = spark.readStream.table("dev.step_right.bronze_orders_valid")
    known_customers = (
        spark.read.table("dev.step_right.bronze_customers_valid")
        .select(col("customer_id").alias("_known_customer_id"))
        .dropDuplicates()
    )
    joined = orders.join(
        known_customers, orders.customer_id == known_customers._known_customer_id, "left"
    )
    critical = {"customer_not_resolved": col("_known_customer_id").isNull()}
    warnings = {
        "missing_order_date": col("order_date").isNull(),
        "invalid_status": ~col("status").isin(KNOWN_STATUSES) & col("status").isNotNull(),
        "future_order_date": col("order_date") > current_date(),
    }
    return tag_business_rules(joined, critical, warnings).drop("_known_customer_id")

@dp.view(name="silver_orders_clean")
def silver_orders_clean():
    return spark.readStream.table("silver_orders_tagged").where("_dq_critical_failed = false")

dp.create_streaming_table("silver_orders_quarantine")

@dp.append_flow(target="silver_orders_quarantine")
def silver_orders_quarantine_flow():
    return spark.readStream.table("silver_orders_tagged").where("_dq_critical_failed = true")

dp.create_streaming_table("silver_orders")

dp.create_auto_cdc_flow(
    target="silver_orders",
    source="silver_orders_clean",
    keys=["order_id"],
    sequence_by="updated_at",
    except_column_list=["_dq_critical_failed", "_dq_critical_failures"],
    stored_as_scd_type=2,
)
```

`silver_orders_clean` is a `@dp.view`, not a `@dp.table` -- it's a pure filter with nothing worth
materializing on its own, existing only so `AUTO CDC`'s `source` argument has a named stream of
critical-passed rows to merge from. `except_column_list` drops the critical-tagging scaffolding
once it's done its job, while `_dq_warnings` isn't listed, so it survives into `silver_orders` as
an ordinary column on every SCD2 version.

## `silver_customers`: no critical rules, same shape

```python
KNOWN_REGIONS = ["NORTHEAST", "SOUTHEAST", "MIDWEST", "WEST"]

@dp.table(name="silver_customers_tagged", comment="customers tagged with business rule outcomes")
def silver_customers_tagged():
    customers = spark.readStream.table("dev.step_right.bronze_customers_valid")
    warnings = {
        "missing_region": col("region").isNull(),
        "invalid_region": ~col("region").isin(KNOWN_REGIONS) & col("region").isNotNull(),
    }
    return tag_business_rules(customers, critical_checks={}, warning_checks=warnings)

@dp.view(name="silver_customers_clean")
def silver_customers_clean():
    return spark.readStream.table("silver_customers_tagged").where("_dq_critical_failed = false")

dp.create_streaming_table("silver_customers")

dp.create_auto_cdc_flow(
    target="silver_customers",
    source="silver_customers_clean",
    keys=["customer_id"],
    sequence_by="updated_at",
    except_column_list=["_dq_critical_failed", "_dq_critical_failures"],
    stored_as_scd_type=2,
)
```

No `_quarantine` table is declared for customers -- with an empty `critical_checks` dictionary,
nothing would ever land in it. Declaring a table that can never receive a row is dead weight in the
pipeline graph, not defensive design.

## `silver_order_items`: two critical rules, two referential joins

```python
@dp.table(name="silver_order_items_tagged", comment="order items tagged with business rule outcomes")
def silver_order_items_tagged():
    items = spark.readStream.table("dev.step_right.bronze_order_items_valid")
    known_products = (
        spark.read.table("dev.step_right.bronze_products_valid")
        .select(col("product_id").alias("_known_product_id")).dropDuplicates()
    )
    known_orders = (
        spark.read.table("dev.step_right.bronze_orders_valid")
        .select(col("order_id").alias("_known_order_id")).dropDuplicates()
    )
    joined = (
        items
        .join(known_products, items.product_id == known_products._known_product_id, "left")
        .join(known_orders, items.order_id == known_orders._known_order_id, "left")
    )
    critical = {
        "invalid_discount_pct": (col("discount_pct") < 0) | (col("discount_pct") > 100),
        "product_not_resolved": col("_known_product_id").isNull(),
    }
    warnings = {
        "missing_quantity": col("quantity").isNull(),
        "order_not_resolved": col("_known_order_id").isNull(),
        "negative_net_price": (col("unit_price") * (1 - col("discount_pct") / 100)) < 0,
        "excessive_quantity": col("quantity") > 10,
    }
    return tag_business_rules(joined, critical, warnings).drop("_known_product_id", "_known_order_id")

@dp.view(name="silver_order_items_clean")
def silver_order_items_clean():
    return spark.readStream.table("silver_order_items_tagged").where("_dq_critical_failed = false")

dp.create_streaming_table("silver_order_items_quarantine")

@dp.append_flow(target="silver_order_items_quarantine")
def silver_order_items_quarantine_flow():
    return spark.readStream.table("silver_order_items_tagged").where("_dq_critical_failed = true")

dp.create_streaming_table("silver_order_items")

dp.create_auto_cdc_flow(
    target="silver_order_items",
    source="silver_order_items_clean",
    keys=["order_item_id"],
    sequence_by="updated_at",
    except_column_list=["_dq_critical_failed", "_dq_critical_failures"],
    stored_as_scd_type=2,
)
```

`order_not_resolved` staying a warning rather than a critical rule is the one rule in this table
that isn't as simple as "bad data, quarantine it" -- flagged in Lecture 1's design and worth
restating here in code: a line item whose parent order hasn't landed yet this micro-batch usually
resolves on the very next run, and quarantining it would just create noisy, self-correcting
"failures" in Section 7's dashboard.

## Verifying SCD2 history and quarantine counts

```sql
-- Current state of every order
SELECT * FROM dev.step_right.silver_orders WHERE __END_AT IS NULL;

-- One order's full history
SELECT order_id, status, __START_AT, __END_AT
FROM dev.step_right.silver_orders
WHERE order_id = 'ORD-0000042'
ORDER BY __START_AT;

-- How many rows were quarantined, and why
SELECT explode(_dq_critical_failures) AS rule, COUNT(*) AS row_count
FROM dev.step_right.silver_order_items_quarantine
GROUP BY rule;

-- Report-only warnings on the current, valid data
SELECT explode(_dq_warnings) AS warning, COUNT(*) AS row_count
FROM dev.step_right.silver_orders
WHERE __END_AT IS NULL
GROUP BY warning;
```

## Common mistakes

- **Filtering after the merge instead of before it.** Applying `WHERE _dq_critical_failed = false`
  against `silver_orders` itself, post-merge, doesn't prevent a bad row from ever being merged --
  it just hides it from that one query. The filter has to happen on `silver_orders_clean`, upstream
  of `create_auto_cdc_flow`.
- **Skipping `dropDuplicates()` on a lookup join, again.** The same bronze-layer mistake from
  Section 2, Lecture 6 fans out row counts here too, on `known_customers`, `known_products`, and
  `known_orders` alike.
- **Forgetting `except_column_list` excludes, not includes.** Leaving `_dq_critical_failed` off
  the list doesn't drop it -- it does the opposite, carrying a column named for failure into every
  row of a table meant to represent validated data.
{: .important }

## What's built so far

Three governed tables with full SCD Type 2 history, built only from business-rule-clean data, with
quarantined rows preserved and report-only warnings riding along for anyone who wants to see them.
Lectures 4 and 5 give the file-sourced side of silver -- `silver_products` -- the same care, minus
the merge machinery these three tables needed.

<!-- prevnext:start -->

---

| [&larr; Previous: Planning and Designing Silver Pipeline for CDC Sources]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-cdc-sources/' | relative_url }}) | [Next: Planning and Designing Silver Pipeline for File Sources &rarr;]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/planning-and-designing-silver-pipeline-for-file-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

