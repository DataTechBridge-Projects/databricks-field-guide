---
title: "Auto Loader With Schema Inference and Evolution"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/auto-loader-with-schema-inference-and-evolution/
read_minutes: 4
---

# Auto Loader With Schema Inference and Evolution
{: .no_toc }

*Estimated read: 4 min*

**Auto Loader** is the ingestion engine underneath the bronze layer of every CDC pipeline in this section -- the Databricks-native answer to "how do I incrementally pick up new files without re-scanning a directory or hand-tracking a watermark." Point it at a cloud storage path with the `cloudFiles` source, and it tracks which files it has already processed using scalable file-notification or directory-listing checkpoints, so a nightly job that used to re-read an entire staging directory now only touches what's new since the last run.

```python
df = (spark.readStream
      .format("cloudFiles")
      .option("cloudFiles.format", "json")
      .option("cloudFiles.schemaLocation", "/checkpoints/orders_bronze/schema")
      .load("/mnt/raw/orders_cdc/"))
```

The part that matters most for a legacy migration is what happens when the *shape* of the incoming data changes. Auto Loader samples the first 50 GB or 1,000 files (whichever comes first) to infer a schema, then persists that schema to the location in `cloudFiles.schemaLocation` so it doesn't have to re-infer on every restart. For loosely typed formats like JSON and CSV it infers everything as `STRING` by default -- deliberately conservative, because guessing `INT` for a column that later contains an alphanumeric order ID is the kind of silent type coercion that corrupts a downstream join. You override that default per column with **schema hints**:

```python
.option("cloudFiles.schemaHints", "order_id STRING, order_date DATE, amount DECIMAL(10,2)")
```

Schema *evolution* is the separate, more consequential question: what happens when a column shows up that wasn't in the original inferred schema. Auto Loader gives you five modes, set via `cloudFiles.schemaEvolutionMode`:

| Mode | Behavior on a new column |
|---|---|
| `addNewColumns` (default) | Stream **fails** once, schema is updated, restart picks up the new column |
| `addNewColumnsWithTypeWidening` | New columns added automatically; compatible type changes (e.g. `int` &rarr; `long`) widened without failing |
| `rescue` | Schema never evolves; anything unexpected is captured in a `_rescued_data` column instead |
| `failOnNewColumns` | Stream fails and stays failed until the schema is manually updated |
| `none` | New columns are silently dropped -- no failure, no rescue |

The default, `addNewColumns`, is intentionally noisy: the first micro-batch that sees a new column stops the stream with an exception rather than silently absorbing it. That's the opposite of how most legacy batch jobs behave -- a Talend job mapping columns positionally would either error obscurely or, worse, shift every downstream field by one. Here, the failure is loud, on purpose, and a simple job restart resumes processing with the new column now part of the schema.

{: .important }
> `rescue` mode is not "safer" than `addNewColumns` -- it's a different tradeoff. It never halts the pipeline, but anything that doesn't match the known schema, including a genuinely new business column your source team just shipped, lands as an opaque JSON blob in `_rescued_data` until someone thinks to look. For a CDC feed carrying business-critical dimension changes, prefer `addNewColumns` (or `failOnNewColumns` for a stricter contract) so schema drift produces a visible failure instead of a quietly growing pile of unparsed rows.

See the [Auto Loader schema inference and evolution guide](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/schema) for the full mode reference, including how nested struct and array types are handled and how `cloudFiles.maxColumns` guards against runaway schema growth. The next lecture takes the bronze table Auto Loader produces here and turns it into a Type 2 dimension using `AUTO CDC`.

<!-- prevnext:start -->

---

| [&larr; Previous: The CDC Architecture End to End]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/the-cdc-architecture-end-to-end/' | relative_url }}) | [Next: Lakeflow Declarative Pipelines for SCD Type 2 &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/lakeflow-declarative-pipelines-for-scd-type-2/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

