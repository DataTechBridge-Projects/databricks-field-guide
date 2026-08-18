---
title: "The Ingestion Pattern Matrix"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/the-ingestion-pattern-matrix/
read_minutes: 3
---

# The Ingestion Pattern Matrix
{: .no_toc }

*Estimated read: 3 min*

Four lectures of individual decisions -- batch vs. streaming, push vs. pull, JDBC vs. Auto Loader,
build vs. buy -- collapse into one artifact that actually gets used during a migration: a matrix
you fill in once per source table in the workload inventory, and refer back to for every ingestion
build ticket after that.

## The matrix

| Source characteristic | Recommended pattern | Why |
|---|---|---|
| Small dimension table, refreshed nightly, low volume | JDBC bulk pull, full overwrite | Simplicity wins when the table is small enough that a full re-read costs nothing |
| Large fact table, append-only, files exported by upstream process | Auto Loader | Incremental, exactly-once, no sustained load on a source system that isn't even queried directly |
| Large fact table, direct JDBC access required, no file export available | JDBC bulk pull with `partitionColumn`, incremental watermark | Accept the source-system load as a known, scheduled cost when no alternative exists |
| High-volume OLTP table, sub-minute latency needed, row-level updates/deletes | Partner CDC tool (Fivetran/Qlik/Arcion) or Lakeflow Connect managed database connector | Log-based capture is the only pattern that delivers this latency without hammering the source |
| SaaS application data (Salesforce, NetSuite, etc.) | Lakeflow Connect SaaS connector or Fivetran | Purpose-built connectors handle API rate limits and pagination that a hand-rolled pull would reimplement badly |
| Semi-structured or schema-drifting files (JSON logs, IoT payloads) | Auto Loader with schema inference/evolution | Schema-on-read in bronze absorbs drift; enforcement moves to silver |

## Applying it to the workload inventory

Every table in the inventory built during **the autopsy** phase already carries the metadata this
matrix needs as columns: approximate size, change frequency, whether the source system supports
direct query access, and downstream latency requirements. Walking the inventory once with this
matrix produces an ingestion assignment per table before a single pipeline gets built -- which
avoids the common migration anti-pattern of defaulting every table to the same pattern (usually
"JDBC bulk, because that's what the first table used") regardless of whether it fits.

```python
# One row of a migration inventory, annotated with the matrix's recommendation
{
    "source_table": "ORDERS",
    "size_gb": 9800,
    "change_frequency": "continuous",
    "direct_query_allowed": False,
    "latency_requirement_minutes": 5,
    "recommended_pattern": "partner_cdc_tool",
}
```

{: .important }
> Treat the matrix as a starting recommendation, not a rule that overrides judgment -- a table that
> looks like a clear Auto Loader candidate on size and volume alone might still need a CDC tool if
> the business requirement is "reflect deletes within one minute," something Auto Loader's
> append-oriented file-watching model doesn't handle on its own.

With every table in the inventory assigned an ingestion pattern, the section's quiz checks that the
underlying decision criteria -- not just the tool names -- have actually landed.

<!-- prevnext:start -->

---

| [&larr; Previous: Partner Tools: Fivetran, Qlik, Arcion for CDC]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/partner-tools-fivetran-qlik-arcion-for-cdc/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

