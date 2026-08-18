---
title: "The 5-Layer Stack: Count, Sum, Checksum, Hash, Semantic"
parent: "The Reconciliation Stack: Proving Semantic Parity"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/the-5-layer-stack-count-sum-checksum-hash-semantic/
read_minutes: 3
---

# The 5-Layer Stack: Count, Sum, Checksum, Hash, Semantic
{: .no_toc }

*Estimated read: 3 min*

The reconciliation stack has five rungs, and the order matters: each layer is cheaper to run and coarser to interpret than the one above it, so a well-built engine runs them in sequence and stops at the first failure rather than burning compute on a hash-level diff before confirming the row counts even agree.

**Layer 1 -- Count.** The cheapest check and the first gate: `SELECT COUNT(*)` from source and target, partitioned by whatever grain the migration is validating per run (a load date, a business unit). A mismatch here means rows were dropped, duplicated, or filtered differently between the legacy extract and the Databricks ingestion path -- go fix the pipeline before running anything more expensive.

**Layer 2 -- Sum.** Aggregate every numeric column that matters financially or operationally: `SUM(order_total)`, `SUM(quantity)`, `SUM(balance)`. This is where the Oracle-`NUMBER`-versus-Spark-`DECIMAL` drift discussed later in this section first surfaces -- two tables with identical row counts can still disagree here by a small but real amount, and sum parity is the layer that catches it before it's buried under millions of individually "close enough" rows.

**Layer 3 -- Checksum.** A single scalar per column (or per table) computed by hashing or summing every value in that column, independent of row order. In Databricks SQL this is commonly built from `sha2()` or `hash()` applied per row, then aggregated:

```sql
SELECT
  SUM(hash(customer_id, order_total, order_status)) AS row_checksum
FROM silver.orders;
```

Checksum tells you *whether* a column-level divergence exists across the whole table without yet telling you *which* rows are responsible -- it's the layer between "something might be wrong" (sum) and "here are the exact rows" (hash).

**Layer 4 -- Hash.** A per-row fingerprint, typically `sha2(concat_ws('|', col1, col2, ...), 256)`, computed identically on both sides and compared row by row on a stable join key. This is the layer that actually localizes a divergence: an `EXCEPT` between the source hash set and the target hash set returns exactly the rows that differ, which turns "reconciliation failed" into a concrete, debuggable list.

```sql
SELECT source_id, source_hash
FROM source_hashes
EXCEPT
SELECT target_id, target_hash
FROM target_hashes;
```

See the Databricks documentation on the [`sha2()` function](https://docs.databricks.com/aws/en/sql/language-manual/functions/sha2) for the hash-length options this pattern relies on.

**Layer 5 -- Semantic.** The layer that isn't a diff at all -- it's a re-execution of business logic against both platforms and a comparison of *outcomes*, not raw values. A customer-tier calculation, a late-fee assessment, a fraud-risk score: run the same business rule on both sides and confirm the classifications agree, even when the underlying numbers technically reconcile at every layer below. This is the only layer that catches a `PL/SQL` procedure's logic being subtly mistranslated into PySpark in a way that produces a *different but plausible* answer rather than an obviously wrong one.

Layers 1 through 4 are fully automatable and cheap to run on every load. Layer 5 requires domain knowledge of what the business logic is supposed to produce, which is why the next two lectures walk through real incidents -- a `ROW_NUMBER` bug and a decimal-sum drift -- that layers 2 through 4 exist specifically to catch.

<!-- prevnext:start -->

---

| [&larr; Previous: Row Counts Lie: Why Semantic Parity Is the Only Truth]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/row-counts-lie-why-semantic-parity-is-the-only-truth/' | relative_url }}) | [Next: The $4M Bug: ROW_NUMBER Sort-Order Differences &rarr;]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/the-4m-bug-row-number-sort-order-differences/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

