---
title: "Row Counts Lie: Why Semantic Parity Is the Only Truth"
parent: "The Reconciliation Stack: Proving Semantic Parity"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/row-counts-lie-why-semantic-parity-is-the-only-truth/
read_minutes: 3
---

# Row Counts Lie: Why Semantic Parity Is the Only Truth
{: .no_toc }

*Estimated read: 3 min*

Every migration project eventually produces the slide that says `SELECT COUNT(*)` matched between Oracle and the lakehouse target, and every migration project has, at some point, shipped that slide while sitting on a production-breaking bug. **Row-count parity** is necessary -- if the counts don't match, stop immediately -- but it is nowhere close to sufficient. A table can have the exact right number of rows and still be wrong in every way that matters to the business: wrong amounts, wrong assignments, wrong history. A legacy DBA who spent years trusting `COUNT(*)` as the reconciliation gate for a same-vendor upgrade (Oracle 11g to 19c, say) is walking into a different kind of migration here, one where the engine itself changed, and the failure modes that engine change introduces don't show up in a row count at all.

Two rows can be numerically identical in quantity and still diverge in ways `COUNT(*)` cannot detect: a `ROW_NUMBER` window function with a non-unique `ORDER BY` can assign row 1 to a different customer in Spark than it did in Oracle, silently reassigning revenue between accounts while the total row count and total revenue both stay identical. A `NUMBER` column summed in Oracle and a `DECIMAL` column summed in Spark can drift by fractions of a cent per row that compound into a real dollar variance at scale, while every individual row still "matches" within a casual glance. Neither of these produces a row-count mismatch. Both produce a wrong answer that a downstream finance or compliance team will eventually notice -- usually after cutover, when rolling back means reconstructing weeks of transactions instead of re-running a validation job.

This is why **semantic parity** -- proving the migrated data means the same thing the source data meant, not just that it has the same shape -- is the only standard a migration team should sign a cutover against. Semantic parity isn't a single check; it's a ladder of checks, each one catching a class of divergence the checks below it structurally cannot see. Row count catches missing or duplicated rows. Sum catches gross numeric drift. Checksum catches column-level corruption. Row-level hash catches which *specific* rows diverge. Semantic validation catches business-rule outputs that differ even when every underlying number technically reconciles -- a rank, a tier assignment, a status flag computed from logic that behaves subtly differently between Oracle's `PL/SQL` and its Spark SQL or PySpark translation.

{: .important }
> Treat row-count parity as a gate you pass through, never a gate you stop at. A matching `COUNT(*)` earns the right to run the next four layers -- it does not, by itself, earn the right to cut over.

The next lecture lays out that full five-layer stack -- count, sum, checksum, hash, and semantic -- as the concrete validation ladder this section builds toward, with each layer defined precisely enough to automate.

<!-- prevnext:start -->

---

| [&larr; Previous: The Reconciliation Stack: Proving Semantic Parity]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/' | relative_url }}) | [Next: The 5-Layer Stack: Count, Sum, Checksum, Hash, Semantic &rarr;]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/the-5-layer-stack-count-sum-checksum-hash-semantic/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

