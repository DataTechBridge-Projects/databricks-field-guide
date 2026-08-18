---
title: "Enforcing Schema at Bronze: Data Contracts"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/enforcing-schema-at-bronze-data-contracts/
read_minutes: 4
---

# Enforcing Schema at Bronze: Data Contracts
{: .no_toc }

*Estimated read: 4 min*

A **data contract** is the formal version of an assumption every legacy ETL pipeline made informally: that the source team wouldn't rename a column, retype a field, or start sending nulls in a `NOT NULL` field without telling you first. In an on-prem warehouse that assumption mostly held, because the source and the warehouse were often owned by the same DBA team, and a schema change to a production Oracle table went through the same change-control process as anything else. Streaming CDC from a cloud-native or third-party source breaks that assumption -- an upstream application team can ship a field rename in a Tuesday deploy with no warehouse team in the loop, and your pipeline finds out when it fails, or worse, when it doesn't.

On Databricks, a data contract is enforced with **expectations** -- declarative quality constraints attached directly to a streaming table or materialized view inside a Lakeflow Declarative Pipeline. Three operators give you three different severities:

| Operator | Behavior on a failing row |
|---|---|
| `expect` | Row is written anyway; failure is recorded as a metric for monitoring |
| `expect_or_drop` | Row is dropped before it's written; pipeline keeps running |
| `expect_or_fail` | Pipeline **stops** and the update rolls back |

```python
import dlt
from pyspark.sql.functions import expr

@dlt.table(name="orders_bronze")
@dlt.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
@dlt.expect_or_drop("valid_amount", "amount >= 0")
@dlt.expect_or_fail("known_operation", "operation IN ('INSERT', 'UPDATE', 'DELETE')")
def orders_bronze():
    return (spark.readStream.table("orders_raw_autoloader"))
```

The choice of operator *is* the contract negotiation. `valid_order_id` and `valid_amount` are drop-level -- a handful of malformed rows shouldn't halt an entire pipeline, but they also shouldn't silently pollute the dimension, so they're quarantined by being dropped (and counted, via the pipeline's event log, so someone can go investigate the source). `known_operation` is fail-level, because an operation code outside the known `INSERT`/`UPDATE`/`DELETE` set means something structural changed upstream -- a new operation type, a corrupted feed, a schema the pipeline was never designed to read -- and continuing to process would risk applying the wrong logic to every subsequent row.

This is the direct Databricks-native counterpart to what a well-run legacy shop did with a staging-table validation script run *after* the load: reject rows that violate a `CHECK` constraint, page someone if referential integrity breaks. The difference is timing and locality -- expectations run inline, as part of the same flow that writes bronze, so a contract violation is caught before a single bad row reaches the `AUTO CDC` flow feeding your SCD Type 2 dimension, not the next morning in a reconciliation report.

{: .important }
> Expectations only apply within Lakeflow Declarative Pipelines, and only to streaming tables, materialized views, and (temporarily) views -- they are not a general Delta Lake feature you can attach to any table. If part of your medallion architecture writes to bronze outside a declarative pipeline (a plain Auto Loader job, say), the contract has to be enforced separately, typically with Delta Lake's native `CHECK` constraints or a validation step in that job.

See the [Lakeflow Declarative Pipelines expectations reference](https://docs.databricks.com/aws/en/dlt/expectations) for combining multiple expectations with `expect_all`, `expect_all_or_drop`, and `expect_all_or_fail`, and for how expectation failures surface in the pipeline's event log and system tables. The next lecture walks through what happens in a real production incident when a contract like this one isn't in place.

<!-- prevnext:start -->

---

| [&larr; Previous: Lakeflow Declarative Pipelines for SCD Type 2]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/lakeflow-declarative-pipelines-for-scd-type-2/' | relative_url }}) | [Next: The Schema-Drift Death Spiral &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/the-schema-drift-death-spiral/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

