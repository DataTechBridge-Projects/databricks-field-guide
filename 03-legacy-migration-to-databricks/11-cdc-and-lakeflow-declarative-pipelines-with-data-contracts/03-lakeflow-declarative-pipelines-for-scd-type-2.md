---
title: "Lakeflow Declarative Pipelines for SCD Type 2"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/lakeflow-declarative-pipelines-for-scd-type-2/
read_minutes: 3
---

# Lakeflow Declarative Pipelines for SCD Type 2
{: .no_toc }

*Estimated read: 3 min*

If you've ever hand-written the **SCD Type 2** pattern in Oracle or SQL Server, you know the shape of the procedure by heart: close out the current row by setting its `end_date` and `is_current` flag, insert a new row with `start_date = today` and `is_current = 'Y'`, and wrap both statements in a transaction so a mid-run failure never leaves the dimension in a half-updated state. It's maybe forty lines of `MERGE` and `UPDATE`, and it's the kind of procedure every warehouse team has copy-pasted and subtly broken at least once -- an off-by-one on the boundary date, a forgotten `WHERE is_current = 'Y'` that updates every historical row instead of just the current one.

Lakeflow Declarative Pipelines replace that hand-written procedure with a single declarative statement: **`AUTO CDC`** (the current name for what was previously called `APPLY CHANGES`). You point it at the bronze stream from the previous lecture, tell it which column identifies a unique business key, which column sequences the changes in order, and whether you want `SCD TYPE 1` (overwrite, current state only) or `SCD TYPE 2` (append, full history) -- and the engine generates the merge logic, including the closing and opening of `__START_AT` / `__END_AT` boundaries, for you.

```python
import dlt

dlt.create_streaming_table("orders_dim_silver")

dlt.create_auto_cdc_flow(
    target="orders_dim_silver",
    source="orders_bronze",
    keys=["order_id"],
    sequence_by="change_timestamp",
    apply_as_deletes="operation = 'DELETE'",
    except_column_list=["operation", "change_timestamp"],
    stored_as_scd_type="2",
)
```

The output columns `__START_AT` and `__END_AT` play the exact role your `start_date`/`end_date` pair played in the legacy dimension -- one open-ended row per business key represents the current version (`__END_AT IS NULL`), and every prior version is retained with its boundary timestamps intact for point-in-time queries. `sequence_by` is what makes this safe against out-of-order delivery: if two changes for the same `order_id` arrive in the wrong order because of network retries or partition skew, `AUTO CDC` uses the sequencing column, not arrival order, to decide which version is actually newer -- a guarantee a naively ordered `MERGE INTO` does not give you for free.

{: .important }
> `AUTO CDC` requires a **streaming table** as its target, not a plain Delta table or a materialized view -- `dlt.create_streaming_table` (or the SQL `CREATE STREAMING TABLE`) must run first. Skipping that step is the most common reason a first `AUTO CDC` flow fails to deploy, and the error message ("target table does not exist" or a type mismatch) doesn't always make the missing streaming-table declaration obvious.

Handling deletes is the other place this diverges from a naive `MERGE`. `apply_as_deletes` tells the flow which incoming rows represent a source-side delete rather than an insert or update; for `SCD TYPE 2`, that closes out the current row's `__END_AT` without inserting a new open-ended version, so the dimension correctly shows "this record existed until it was deleted" rather than manufacturing a phantom final state. See the [`AUTO CDC` reference](https://docs.databricks.com/aws/en/dlt/cdc) for the full syntax, including `AUTO CDC FROM SNAPSHOT` for sources that only ever give you full snapshots rather than a true change feed, and multi-column sequencing for sources whose ordering key isn't a single timestamp.

The next lecture backs up one layer, to the bronze boundary this flow reads from, and covers how to stop a malformed or unexpected change from ever reaching `AUTO CDC` in the first place.

<!-- prevnext:start -->

---

| [&larr; Previous: Auto Loader With Schema Inference and Evolution]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/auto-loader-with-schema-inference-and-evolution/' | relative_url }}) | [Next: Enforcing Schema at Bronze: Data Contracts &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/enforcing-schema-at-bronze-data-contracts/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

