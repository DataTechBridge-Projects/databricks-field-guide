---
title: "Tolerance Thresholds: Acceptable vs Real Variance"
parent: "The Reconciliation Stack: Proving Semantic Parity"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/tolerance-thresholds-acceptable-vs-real-variance/
read_minutes: 3
---

# Tolerance Thresholds: Acceptable vs Real Variance
{: .no_toc }

*Estimated read: 3 min*

A reconciliation engine that demands exact-to-the-penny equality on every run will eventually fail a load that is, in every way that matters, correct -- and a team that gets paged for a false alarm often enough starts ignoring the pages. A reconciliation engine that accepts any variance under a loose threshold will eventually wave through a real bug. The job of a **tolerance threshold** is to sit precisely between those two failure modes, and getting it there takes an explicit decision about what kind of variance each layer of the stack is allowed to absorb.

Start from the two prior lectures. The `DECIMAL`-versus-`DOUBLE` lecture established that even correctly typed `DECIMAL` columns can carry a few fractions of a cent of genuine, unavoidable rounding-mode difference between Oracle's and Spark's arithmetic engines on large aggregates -- that variance is real, expected, and not a bug. The `ROW_NUMBER` lecture established that a tie-break ordering difference is *not* the kind of variance any threshold should absorb -- it produces a discrete misassignment, not a small numeric drift, and no tolerance setting makes a wrongly-ranked row acceptable. The distinction a reconciliation engine has to encode is between **continuous, bounded numeric variance** (tolerable, set a threshold) and **discrete logical divergence** (never tolerable, must be zero).

A practical tolerance policy sets thresholds per layer, not one global number:

| Layer | Tolerance approach |
|---|---|
| Count | Zero tolerance -- row counts must match exactly, always |
| Sum | Relative tolerance, e.g. `ABS(source_sum - target_sum) / source_sum < 0.0001` (0.01%), tuned per table by transaction volume |
| Checksum | Zero tolerance -- a checksum mismatch means investigate, not average away |
| Hash | Zero tolerance -- any row-level hash mismatch names a specific row to review |
| Semantic | Zero tolerance on the classification itself, but the upstream numeric inputs to that classification inherit the sum layer's tolerance |

The sum layer is the only one that typically warrants a nonzero threshold, and even there the right number is not a guess -- it should be derived from the actual magnitude of expected rounding variance for that table's row count and value distribution, then set with headroom, not set arbitrarily loose to make alerts stop firing. A threshold of 0.01% on a table with ten million dollar-denominated rows still catches a six-figure discrepancy while absorbing the few cents of legitimate rounding drift a `DECIMAL` migration produces.

```sql
SELECT
  s.table_name,
  s.total AS source_total,
  t.total AS target_total,
  ABS(s.total - t.total) / s.total AS relative_variance,
  CASE WHEN ABS(s.total - t.total) / s.total < 0.0001
       THEN 'PASS' ELSE 'FAIL' END AS reconciliation_status
FROM source_sums s
JOIN target_sums t ON s.table_name = t.table_name;
```

{: .important }
> Never set a tolerance threshold to make a specific failing check pass. Derive the threshold from the expected magnitude of legitimate rounding variance for that table, document the reasoning, and treat any later change to that threshold as a decision serious enough to require the same sign-off as the original reconciliation gate.

This closes the reconciliation-stack theory covered in this section -- five layers, a tie-break discipline for window functions, a `DECIMAL`-first typing rule, and a tolerance policy that distinguishes acceptable drift from a real bug. The next section, Building the Reconciliation Engine, turns this into a running PySpark job with dashboards and a reusable script library.

<!-- prevnext:start -->

---

| [&larr; Previous: Floating-Point Variance: SUM(DECIMAL) Oracle vs Spark]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/floating-point-variance-sum-decimal-oracle-vs-spark/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

