---
title: "Developing Bronze Pipeline for CDC Sources"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-pipeline-for-cdc-sources/
read_minutes: 19
---

# Developing Bronze Pipeline for CDC Sources
{: .no_toc }

*Estimated read: 19 min*

From plan to running pipeline. This lecture builds `transformations/bronze_cdc.py` -- fixed
schemas, a shared read helper, and three bronze streaming tables -- deploys it, and inspects the
result.

## Fixed schemas

```python
# transformations/schemas.py
from pyspark.sql.types import (
    StructType, StructField, StringType, DoubleType, IntegerType, TimestampType
)

CUSTOMERS_SCHEMA = StructType([
    StructField("customer_id", StringType(), nullable=False),
    StructField("first_name", StringType(), nullable=True),
    StructField("last_name", StringType(), nullable=True),
    StructField("email", StringType(), nullable=True),
    StructField("region", StringType(), nullable=True),
    StructField("signup_date", StringType(), nullable=True),
    StructField("updated_at", TimestampType(), nullable=False),
])

ORDERS_SCHEMA = StructType([
    StructField("order_id", StringType(), nullable=False),
    StructField("customer_id", StringType(), nullable=True),
    StructField("order_date", StringType(), nullable=True),
    StructField("status", StringType(), nullable=True),
    StructField("updated_at", TimestampType(), nullable=False),
])

ORDER_ITEMS_SCHEMA = StructType([
    StructField("order_item_id", StringType(), nullable=False),
    StructField("order_id", StringType(), nullable=True),
    StructField("product_id", StringType(), nullable=True),
    StructField("quantity", IntegerType(), nullable=True),
    StructField("unit_price", DoubleType(), nullable=True),
    StructField("discount_pct", DoubleType(), nullable=True),
])
```

Every column from Lecture 5's seed generator is represented, typed explicitly rather than
inferred -- `customer_id` and `order_id` stay nullable at this schema level because Lecture 1's
plan pushed *validating* those foreign keys to the quality pipeline in Lectures 5-6, not into the
schema itself. A schema controls shape; it doesn't enforce business meaning.

## The shared read helper

```python
# transformations/helpers.py
from pyspark.sql.functions import current_timestamp

def read_cdc_bronze(spark, source_table, schema):
    """Read a raw CDC change feed table with a fixed schema and ingestion metadata."""
    return (
        spark.readStream
        .table(f"dev.raw_cdc.{source_table}")
        .select([field.name for field in schema.fields])
        .withColumn("_ingested_at", current_timestamp())
    )
```

Selecting explicitly by the schema's field names -- rather than trusting whatever columns happen
to be in `dev.raw_cdc.<source_table>` -- is what actually enforces "fixed schema" here: an upstream
column rename shows up as a missing-column error at this `.select()`, immediately, instead of
silently passing through.

## The three bronze tables

```python
# transformations/bronze_cdc.py
import dlt as dp
from schemas import CUSTOMERS_SCHEMA, ORDERS_SCHEMA, ORDER_ITEMS_SCHEMA
from helpers import read_cdc_bronze

@dp.table(
    name="bronze_customers",
    comment="Raw customer CDC feed, fixed schema, no business validation"
)
def bronze_customers():
    return read_cdc_bronze(spark, "customers_changefeed", CUSTOMERS_SCHEMA)

@dp.table(
    name="bronze_orders",
    comment="Raw order CDC feed, fixed schema, no business validation"
)
def bronze_orders():
    return read_cdc_bronze(spark, "orders_changefeed", ORDERS_SCHEMA)

@dp.table(
    name="bronze_order_items",
    comment="Raw order item CDC feed, fixed schema, no business validation"
)
def bronze_order_items():
    return read_cdc_bronze(spark, "order_items_changefeed", ORDER_ITEMS_SCHEMA)
```

Three tables, three calls to the same helper, no repeated read logic -- exactly the shape Lecture 1
planned. Each function's only job is naming which source table and which schema it reads with;
`read_cdc_bronze` does the actual work.

## Pipeline configuration

```yaml
# pipeline.yml
name: steprightproject-bronze-cdc
catalog: dev
schema: step_right
serverless: true
continuous: false
libraries:
  - glob:
      include: transformations/**
```

Same shape as the `pipeline.yml` pattern from Part 1, Section 9 -- serverless compute, triggered
(not continuous) mode, since this pipeline is meant to be kicked off by a Lakeflow Job run
(Section 5), not to run indefinitely watching for changes.

## Deploying and running

```bash
databricks pipelines create --json @pipeline.yml
databricks pipelines start --pipeline-id <pipeline-id>
```

Or through the workspace UI: **Workflows -> Pipelines -> Create pipeline**, pointing at
`pipeline.yml`. Either way, the pipeline graph view should show three independent nodes --
`bronze_customers`, `bronze_orders`, `bronze_order_items` -- with no edges between them, since none
of these tables reference each other yet. That changes in Section 3, where silver reads all three.

## Verifying the load

```sql
SELECT COUNT(*) FROM dev.step_right.bronze_customers;
SELECT COUNT(*) FROM dev.step_right.bronze_orders;
SELECT COUNT(*) FROM dev.step_right.bronze_order_items;

SELECT * FROM event_log(TABLE(dev.step_right.bronze_orders))
WHERE event_type = 'flow_progress'
ORDER BY timestamp DESC
LIMIT 10;
```

Row counts should roughly match batch zero's generation targets from Section 1, Lecture 4 --
around 2,000 customers, 12,000 orders, and somewhere between 12,000 and 48,000 order items,
depending on how many line items each generated order got. If a count is far off, check the loader
notebook actually ran against `batch_zero` before checking the pipeline itself.

## Common first-run mistakes

- **Forgetting `.select()` on the schema's field list.** Without it, `read_cdc_bronze` silently
  passes through every column in the source table, including any extra column a future CDC
  connector version adds -- defeating the entire point of a fixed schema.
- **A `TimestampType` mismatch on `updated_at`.** If the raw change feed table stores `updated_at`
  as a string (as Section 1, Lecture 5's loader notebook does, since it writes from JSON), casting
  happens implicitly on `.select()` -- but a genuinely malformed timestamp string fails the whole
  microbatch rather than one row. This is exactly the "fail loudly" behavior Lecture 1 chose on
  purpose; it's a signal to look at the quarantine pattern in Lectures 5-6, not to loosen the
  schema.
- **Running this pipeline before the loader notebook.** An empty `dev.raw_cdc` table produces
  three empty bronze tables with no error at all -- a streaming read against an empty source isn't
  a failure. Always verify row counts in `dev.raw_cdc` first if bronze looks suspiciously empty.
{: .important }

## What's built so far

Three governed, structurally sound bronze tables for StepRight's core operational entities, with
zero duplicated read logic and a schema contract that fails fast on upstream drift. Lectures 3 and
4 build the equivalent for the file-based sources; Lectures 5 and 6 add the quality tagging layer
these tables -- and the file-based ones -- both need before silver can trust them.

<!-- prevnext:start -->

---

| [&larr; Previous: Planning and Designing Bronze Pipeline for CDC Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-cdc-sources/' | relative_url }}) | [Next: Planning and Designing Bronze Pipeline for File Sources &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-file-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

