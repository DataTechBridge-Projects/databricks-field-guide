---
title: "Architecting the Reconciliation Job"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/architecting-the-reconciliation-job/
read_minutes: 3
---

# Architecting the Reconciliation Job
{: .no_toc }

*Estimated read: 3 min*

Before writing a line of PySpark, decide what the reconciliation job is actually going to prove and where the proof lives. The prior section's five-layer stack is the *what*; this lecture is the *how* -- the architecture that runs those layers on a schedule and leaves a durable record behind, rather than a one-off notebook run someone screenshots for a sign-off email.

Collapse the five layers into three **parity levels** for the job's design, since count and sum are cheap enough to run inline as a pre-check, and checksum and hash do the same real work at different granularities:

- **Count parity** -- row totals per partition (load date, source system, business unit). The fast gate that runs first and aborts the rest of the job on failure, so a dropped-rows bug doesn't waste compute computing hashes for data that was never going to match.
- **Hash parity** -- the per-row `sha2()` fingerprint from the prior section, joined on a stable business key between source and target. This is where individual divergent rows get identified, not just detected.
- **Numeric tolerance** -- for financial and operational sums, "equal" has to mean "within a defined tolerance," not bit-for-bit identical, because Oracle `NUMBER` and Spark `DECIMAL` round differently at the edges. Baking a tolerance threshold into the job now avoids a later argument about whether a one-cent difference on a billion-row table is a real bug.

The architectural decision that matters most is where results go. Every run should write its output -- not just pass/fail, but row-level detail on every mismatch -- into a **Delta audit table**, append-only, partitioned by run date and table pair:

```python
audit_row = (
    spark.createDataFrame(mismatches)
    .withColumn("run_ts", F.current_timestamp())
    .withColumn("table_pair", F.lit("legacy.orders -> silver.orders"))
    .withColumn("layer", F.lit("hash"))
)
audit_row.write.format("delta").mode("append").saveAsTable("recon.audit_log")
```

{: .important }
Treat the audit table as a permanent trust ledger, not scratch output -- never overwrite or truncate it. The value of reconciliation compounds over time: six months after cutover, "show me every parity check this table has ever passed" is the artifact that ends an audit or a data-quality dispute in one query, and it only exists if nothing has ever cleared it.

Structure the job as a parameterized Lakeflow Job task -- source table, target table, join key, and tolerance passed in as parameters -- so the same code runs against every table pair in the migration inventory rather than being copy-pasted per table. The next four lectures build out each piece of this architecture: the PySpark hash-diff logic itself, the dashboard that reads the audit table, the case for running this nightly starting on day one, and the config-driven library that makes one engine reusable across the whole migration.

<!-- prevnext:start -->

---

| [&larr; Previous: Building the Reconciliation Engine]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/' | relative_url }}) | [Next: The PySpark Hash-Diff Implementation &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-pyspark-hash-diff-implementation/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

