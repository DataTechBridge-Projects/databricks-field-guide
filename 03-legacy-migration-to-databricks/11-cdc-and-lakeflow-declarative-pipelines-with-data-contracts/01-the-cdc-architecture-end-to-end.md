---
title: "The CDC Architecture End to End"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/the-cdc-architecture-end-to-end/
read_minutes: 3
---

# The CDC Architecture End to End
{: .no_toc }

*Estimated read: 3 min*

Your legacy warehouse almost certainly already does **change data capture (CDC)** in some form -- Oracle GoldenGate shipping redo-log entries, SQL Server Change Tracking, or a Talend job diffing a source table against yesterday's snapshot. What changes on Databricks is not the concept, it's where each step of that pipeline lives and how much of it you have to hand-write versus declare. This lecture walks the whole path end to end before the next five lectures zoom into each stage.

A CDC pipeline has four jobs, in order: **capture** the change (an insert, update, or delete) at the source; **land** it somewhere durable without losing or duplicating events; **apply** it to a target table that represents current or historical state; and **enforce** that a malformed or unexpected change doesn't corrupt that target silently. In a legacy ETL stack those four jobs are usually four different tools stitched together with cron and hope -- a log-shipping agent, a staging table, a hand-written `MERGE` procedure, and whatever validation someone remembered to add. On Databricks, the same four jobs map onto a much shorter chain:

| Legacy stage | Databricks equivalent |
|---|---|
| Log-shipping agent / CDC connector (GoldenGate, Debezium) | **Lakeflow Connect** or a **CDC connector landing files/events** into cloud storage |
| Staging table, manually truncated and reloaded | **Auto Loader** (`cloudFiles`) incrementally ingesting new files into a bronze Delta table |
| Hand-written `MERGE INTO` + effective-dating logic | **`AUTO CDC`** inside a Lakeflow Declarative Pipeline, targeting a streaming table |
| Validation script, often run *after* the load already committed | **Data contract** enforced at bronze, before a bad row ever reaches silver |

The critical architectural shift is *where enforcement happens*. A legacy batch job typically loads first and validates second -- a reconciliation query run the next morning catches problems after they've already propagated. A Lakeflow Declarative Pipeline can enforce a contract inline, as part of the same flow that writes the row, so a schema violation stops the pipeline before it lands rather than getting caught in tomorrow's audit.

End to end, the flow looks like this: a source change lands as a file or event in cloud storage; **Auto Loader** picks it up incrementally, infers or validates its schema, and writes it to a bronze Delta table with the raw change payload intact (including an `operation` column and a sequencing column such as a timestamp or LSN); a Lakeflow Declarative Pipeline reads that bronze stream and applies `AUTO CDC` to produce a silver table -- either a Type 1 "current state only" dimension or a Type 2 dimension carrying full history; and gold-layer aggregates build on top of silver exactly as they would in any medallion design.

{: .important }
> The single most common migration mistake in this chain is treating Auto Loader as "just a file reader" and skipping schema enforcement, because the legacy staging table never needed it -- a fixed-width mainframe extract or a well-governed Oracle export rarely changed shape underneath the job. A cloud-native source (an app team's event stream, a SaaS API) changes shape far more often, and without a contract at bronze, that change reaches your dimension table before anyone notices.

Each of the next four lectures takes one link in this chain -- Auto Loader's schema behavior, `AUTO CDC` for SCD Type 2, contract enforcement at bronze, and what happens when that enforcement is missing -- and goes deep enough to build and defend the pattern in a real migration.

<!-- prevnext:start -->

---

| [&larr; Previous: CDC and Lakeflow Declarative Pipelines With Data Contracts]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/' | relative_url }}) | [Next: Auto Loader With Schema Inference and Evolution &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/auto-loader-with-schema-inference-and-evolution/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

