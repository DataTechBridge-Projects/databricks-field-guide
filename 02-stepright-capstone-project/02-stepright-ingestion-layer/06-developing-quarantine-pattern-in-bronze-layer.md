---
title: "Developing Quarantine Pattern in Bronze Layer"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 6
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/developing-quarantine-pattern-in-bronze-layer/
read_minutes: 13
---

# Developing Quarantine Pattern in Bronze Layer
{: .no_toc }

*Estimated read: 13 min*

Lecture 5 designed tag-don't-drop quality tagging and a valid/quarantine split, all downstream of
untouched raw bronze. This lecture implements it -- a reusable tagging helper, one fully worked
example with a real referential check, and the fan-out pattern that actually splits valid rows
from quarantined ones.

## A reusable tagging helper

```python
# transformations/dq_helpers.py
from pyspark.sql.functions import when, lit, array, array_except, size, col

def tag_quality(df, checks: dict):
    """
    checks: {rule_name: boolean column that is TRUE when the row FAILS that rule}
    Adds _dq_failed_rules (array of failed rule names) and _dq_valid (bool).
    """
    failed_flags = array(*[
        when(condition, lit(name)) for name, condition in checks.items()
    ])
    return (
        df
        .withColumn("_dq_failed_rules", array_except(failed_flags, array(lit(None).cast("string"))))
        .withColumn("_dq_valid", size(col("_dq_failed_rules")) == 0)
    )
```

Every quality-tagged table in this pipeline calls `tag_quality` with a dictionary of named
checks -- the table below only needs to define *what* to check, not how tagging itself works.

## Worked example: `bronze_orders` with a referential check

`bronze_orders` needs one check `tag_quality` can't do alone: whether `customer_id` actually
exists in `bronze_customers`. That's a **stream-static join** -- the CDC stream of orders joined
against a batch snapshot of known customer IDs -- a standard, supported Structured Streaming
pattern for exactly this "does this foreign key resolve" question:

```python
# transformations/bronze_quality.py
import dlt as dp
from pyspark.sql.functions import col
from dq_helpers import tag_quality

@dp.table(name="bronze_orders_tagged", comment="bronze_orders with structural DQ tags")
def bronze_orders_tagged():
    orders = spark.readStream.table("dev.step_right.bronze_orders")
    known_customers = (
        spark.read.table("dev.step_right.bronze_customers")
        .select(col("customer_id").alias("_known_customer_id"))
        .dropDuplicates()
    )
    joined = orders.join(
        known_customers, orders.customer_id == known_customers._known_customer_id, "left"
    )
    checks = {
        "missing_order_id": col("order_id").isNull(),
        "unknown_customer_id": col("_known_customer_id").isNull() & col("customer_id").isNotNull(),
    }
    return tag_quality(joined, checks).drop("_known_customer_id")
```

The left join is what makes `unknown_customer_id` answerable: if `customer_id` doesn't match any
row in `known_customers`, `_known_customer_id` comes back null after the join, and the check flags
it -- exactly the ~1% of orders Section 1, Lecture 5's generator deliberately pointed at
`CUST-999999`, a customer ID that was never generated.

## The fan-out: splitting valid from quarantined

SDP's **append flow** pattern is what actually performs the split -- two independent streaming
writes into two different target tables, both reading from the same tagged source:

```python
dp.create_streaming_table("bronze_orders_valid")
dp.create_streaming_table("bronze_orders_quarantine")

@dp.append_flow(target="bronze_orders_valid")
def bronze_orders_valid_flow():
    return spark.readStream.table("bronze_orders_tagged").where("_dq_valid = true")

@dp.append_flow(target="bronze_orders_quarantine")
def bronze_orders_quarantine_flow():
    return spark.readStream.table("bronze_orders_tagged").where("_dq_valid = false")
```

`bronze_orders_valid` is what Section 3's silver pipeline reads from. `bronze_orders_quarantine`
keeps every rejected row, with `_dq_failed_rules` explaining exactly why, for Section 7's
monitoring dashboard.

## Generating the simpler tables from the same helper

`bronze_customers` and `bronze_products` only need null checks, no joins -- a good case for the
same factory-function approach Lecture 4 used for the file sources:

```python
SIMPLE_QUALITY_TABLES = {
    "bronze_customers": {"missing_customer_id": col("customer_id").isNull()},
    "bronze_products": {
        "missing_product_id": col("product_id").isNull(),
        "missing_list_price": col("list_price").isNull(),
    },
}

def make_quality_tables(source_name: str, checks: dict):
    @dp.table(name=f"{source_name}_tagged", comment=f"{source_name} with structural DQ tags")
    def _tagged():
        return tag_quality(spark.readStream.table(f"dev.step_right.{source_name}"), checks)

    dp.create_streaming_table(f"{source_name}_valid")
    dp.create_streaming_table(f"{source_name}_quarantine")

    @dp.append_flow(target=f"{source_name}_valid")
    def _valid_flow():
        return spark.readStream.table(f"{source_name}_tagged").where("_dq_valid = true")

    @dp.append_flow(target=f"{source_name}_quarantine")
    def _quarantine_flow():
        return spark.readStream.table(f"{source_name}_tagged").where("_dq_valid = false")

for source_name, checks in SIMPLE_QUALITY_TABLES.items():
    make_quality_tables(source_name, checks)
```

`bronze_order_items` follows `bronze_orders`' pattern exactly, just with two referential joins
(`order_id` against `bronze_orders`, `product_id` against `bronze_products`) instead of one, plus
the value checks from Lecture 5's design table (`unit_price >= 0`, `quantity > 0`) -- the same
`tag_quality` helper and append-flow split, only the `checks` dictionary and join logic grow.

## Verifying quarantine counts

```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN size(_dq_failed_rules) = 0 THEN 1 ELSE 0 END) AS valid,
       SUM(CASE WHEN size(_dq_failed_rules) > 0 THEN 1 ELSE 0 END) AS quarantined
FROM dev.step_right.bronze_orders_tagged;

SELECT explode(_dq_failed_rules) AS rule, COUNT(*) AS row_count
FROM dev.step_right.bronze_orders_quarantine
GROUP BY rule
ORDER BY row_count DESC;
```

Against batch zero, `bronze_orders_quarantine` should land close to Section 1, Lecture 5's ~1%
`unknown_customer_id` injection rate, and `bronze_order_items_quarantine` close to its ~2% orphaned
`product_id` rate plus a much smaller share of negative-`unit_price` rows. Numbers well outside
those ranges usually mean the join key or the generator's random seed changed somewhere between
lectures -- worth checking before assuming the tagging logic itself is wrong.

## Common mistakes

- **Joining on the wrong side.** A `right` or `inner` join instead of `left` in the referential
  check silently drops the very rows you're trying to flag -- an inner join between `orders` and
  `known_customers` never produces a null `_known_customer_id` to detect, because unmatched rows
  are removed from the result before the check ever runs.
- **Forgetting `dropDuplicates()` on the lookup side.** A batch read of `bronze_customers` with a
  duplicate `customer_id` (from CDC replaying an update) fans the join out, inflating row counts
  in the tagged table without changing which rows are actually valid.
- **Checking `_dq_valid` before `tag_quality` finishes.** `array_except` needs its second argument
  cast to match the array's element type exactly (`array(lit(None).cast("string"))`) -- omitting
  the cast produces a type-mismatch error that has nothing to do with the actual quality logic.
{: .important }

## What Section 2 leaves ready for Section 3

Seven raw bronze tables, untouched. Seven tagged tables explaining exactly why each row passed or
failed. Seven pairs of valid/quarantine tables, with `_valid` as the only thing silver is allowed
to read. Section 3 picks up from `bronze_orders_valid`, `bronze_customers_valid`, and
`bronze_order_items_valid` to build AUTO CDC's SCD Type 2 history -- structurally sound data,
before a single business rule has been applied to it.

<!-- prevnext:start -->

---

| [&larr; Previous: Design Source Data Quality Monitoring in Bronze Layer]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/design-source-data-quality-monitoring-in-bronze-layer/' | relative_url }}) | [Next: StepRight - Transformation Layers &rarr;]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

