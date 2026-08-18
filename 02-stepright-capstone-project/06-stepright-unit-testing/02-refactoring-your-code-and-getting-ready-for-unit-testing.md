---
title: "Refactoring Your Code and Getting Ready for Unit Testing"
parent: "StepRight - Unit Testing"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/06-stepright-unit-testing/refactoring-your-code-and-getting-ready-for-unit-testing/
read_minutes: 7
---

# Refactoring Your Code and Getting Ready for Unit Testing
{: .no_toc }

*Estimated read: 7 min*

Lecture 1 named two pieces of logic that can't be unit tested as written: `gold_daily_revenue`'s
inline revenue math and `silver_products`'s inline dedup. This lecture pulls both out into
standalone functions -- without changing what either pipeline actually produces.

## The refactoring pattern: separate "what to compute" from "where the data comes from"

Every extraction in this lecture follows the same shape: the decorated `@dp.table` /
`@dp.materialized_view` function keeps doing I/O (`spark.read.table(...)`) and nothing else; a new,
plain function takes the DataFrames that I/O produced as arguments and does the actual
transformation, with a `return` and no table references anywhere inside it. The decorated function
becomes a thin wrapper -- read, then delegate.

## Extracting `gold_daily_revenue`'s calculation

**Before** (Section 4, Lecture 1 -- one function doing both I/O and math):

```python
@dp.materialized_view(name="gold_daily_revenue", comment="...")
def gold_daily_revenue():
    order_items = spark.read.table("silver_order_items").filter("__END_AT IS NULL")
    orders = spark.read.table("silver_orders").filter("__END_AT IS NULL").filter("status != 'CANCELLED'")
    customers = spark.read.table("silver_customers").filter("__END_AT IS NULL")
    products = spark.read.table("silver_products")

    priced = (
        order_items.join(orders, "order_id").join(customers, "customer_id").join(products, "product_id")
        .withColumn("gross_revenue", col("unit_price") * col("quantity"))
        .withColumn("discount_amount", col("gross_revenue") * (col("discount_pct") / 100))
        .withColumn("net_revenue", col("gross_revenue") - col("discount_amount"))
    )
    return priced.groupBy(...).agg(...)
```

**After** (`transformations/gold_logic.py` -- new, pure):

```python
# transformations/gold_logic.py
from pyspark.sql.functions import col, sum as spark_sum, count

def compute_daily_revenue(order_items, orders, customers, products):
    """Pure: four DataFrames in, one aggregated DataFrame out. No spark.read anywhere."""
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

**After** (`transformations/gold_reporting.py` -- now a thin wrapper):

```python
import dlt as dp
from gold_logic import compute_daily_revenue

@dp.materialized_view(name="gold_daily_revenue", comment="...")
def gold_daily_revenue():
    order_items = spark.read.table("silver_order_items").filter("__END_AT IS NULL")
    orders = spark.read.table("silver_orders").filter("__END_AT IS NULL").filter("status != 'CANCELLED'")
    customers = spark.read.table("silver_customers").filter("__END_AT IS NULL")
    products = spark.read.table("silver_products")
    return compute_daily_revenue(order_items, orders, customers, products)
```

Every filter Section 4 was careful about (`__END_AT IS NULL`, `status != 'CANCELLED'`) stays where
it was, on the I/O side -- the extraction moves the *math*, not the correctness-critical filtering
decisions that belong to reading the source tables.

## Extracting `silver_products`'s dedup logic

Same pattern, applied to the window-function dedup from Section 3, Lecture 5:

```python
# transformations/silver_logic.py
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col

def dedupe_latest(df, partition_col, order_col):
    """Pure: keep exactly one row per partition_col, the one with the max order_col."""
    window = Window.partitionBy(partition_col).orderBy(col(order_col).desc())
    return (
        df.withColumn("_rn", row_number().over(window))
        .filter(col("_rn") == 1)
        .drop("_rn")
    )
```

Generalizing to `partition_col` and `order_col` arguments, rather than hardcoding `product_id` and
`_ingested_at`, is a deliberate small step beyond a pure copy-paste extraction -- `silver_products`
calls it as `dedupe_latest(df, "product_id", "_ingested_at")`, but the same function is now
reusable the moment a second file-sourced table ever needs identical "latest wins" logic, without
a second near-duplicate window-function block.

**After** (`transformations/silver_files.py` -- now a thin wrapper, same shape as `gold_reporting.py`'s):

```python
import dlt as dp
from silver_logic import dedupe_latest
from dq_helpers import tag_business_rules

@dp.table(name="silver_products", comment="Deduplicated, validated current view of the product catalog")
def silver_products():
    deduped = dedupe_latest(
        spark.read.table("dev.step_right.bronze_products_valid"), "product_id", "_ingested_at"
    )
    warnings = {
        "invalid_list_price": col("list_price") <= 0,
        "unknown_category": ~col("category").isin(KNOWN_CATEGORIES),
    }
    return tag_business_rules(deduped, critical_checks={}, warning_checks=warnings)
```

The tagging step stays inline here rather than moving into `silver_logic.py` -- it's already a call
to a pure, already-tested helper (`tag_business_rules`), so wrapping it in a second layer of
indirection would add a file without adding any actual testability.

## Why this refactor was cheap

Neither extraction required restructuring how StepRight's pipelines are organized, adding a new
dependency, or touching a single table name, schema, or SQL statement Sections 2-4 already
verified by hand. That's not luck -- it's the payoff of `tag_quality` and `tag_business_rules`
having already established the DataFrame-in/DataFrame-out shape as this project's norm from
Section 2 onward, even before "unit testing" was an explicit goal. A codebase that already leans on
small, composable helper functions refactors toward testability in an afternoon; one where every
transformation is one long inline block, mixing I/O and business logic freely, would have needed a
far larger rewrite to reach the same starting line Lecture 3 now begins from.

## What doesn't move

`tag_quality` and `tag_business_rules` needed no refactoring at all -- they were already pure the
day Section 2 and Section 3 wrote them. Nothing about the pipeline's *behavior* changes here: same
tables, same columns, same values, verified by rerunning Section 4's existing verification queries
against `steprightproject-silver-gold` after deploying the refactored files.

## Freezing known-good output before refactoring anything

Before extracting a single function, capture what the *current*, unrefactored pipeline actually
produces -- a snapshot to diff the refactored version against, rather than trusting "the code looks
equivalent" alone:

```sql
CREATE TABLE dev.step_right._pre_refactor_gold_daily_revenue AS
SELECT * FROM dev.step_right.gold_daily_revenue;
```

After deploying the refactored `gold_reporting.py` and letting `steprightproject-silver-gold`
rebuild `gold_daily_revenue`, a diff against that snapshot is a direct, mechanical answer to "did
the refactor change anything," independent of how carefully the extraction was read by eye:

```sql
SELECT * FROM dev.step_right.gold_daily_revenue
EXCEPT
SELECT * FROM dev.step_right._pre_refactor_gold_daily_revenue;
-- Should return zero rows
```

Drop the snapshot table once the diff comes back empty -- it's a temporary verification step, not
a permanent fixture of the schema.

## One file at a time, not one giant refactor commit

Extract `gold_logic.py` first, deploy, diff against the frozen snapshot, and commit -- then repeat
the same sequence for `silver_logic.py` as a second, independent commit. Two small, independently
verifiable changes are easier to review, and easier to revert individually if one diff surfaces an
unexpected difference, than one large commit touching both pipelines' files at once, where a
regression in either one is harder to isolate from the diff alone.

## Common mistakes

- **Extracting the function but leaving a `spark.read.table` call inside it "just in case."** A
  function with even one table reference isn't pure anymore -- the whole point of the extraction is
  a function pytest can call directly with an in-memory DataFrame, no workspace required.
- **Changing behavior while refactoring.** Renaming a column, reordering a `groupBy`, or "cleaning
  up" a filter while extracting a function turns a should-be-safe refactor into an unverified
  behavior change wearing a refactor's disguise -- extract first, verify identical output, and only
  then consider any actual logic changes as a separate, deliberate step.
{: .important }

## What's next

`gold_logic.py` and `silver_logic.py` now hold pure, testable functions. Lecture 3 writes the
actual `pytest` test cases against them, starting with a local Spark fixture in `conftest.py`.

<!-- prevnext:start -->

---

| [&larr; Previous: Design Test Strategy for the Project]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/design-test-strategy-for-the-project/' | relative_url }}) | [Next: Designing and Developing Unit Test Cases &rarr;]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/designing-and-developing-unit-test-cases/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

