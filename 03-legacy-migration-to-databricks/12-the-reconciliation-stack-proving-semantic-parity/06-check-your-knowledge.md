---
title: "Check Your Knowledge"
parent: "The Reconciliation Stack: Proving Semantic Parity"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of semantic parity, the five-layer reconciliation stack, `ROW_NUMBER` tie-breaks, decimal drift, and tolerance thresholds before moving on.

1. Why is row-count parity alone insufficient to sign off on a migration cutover?
   - A. It is too computationally expensive to run on every load
   - B. Two tables can have identical row counts while individual rows or aggregates are still wrong
   - C. `COUNT(*)` is not supported on Delta tables
   - D. Row counts only work on tables under one million rows

2. In the five-layer reconciliation stack, why should the layers run in order from count through semantic rather than starting with the most detailed check?
   - A. Databricks requires checks to run in a fixed order
   - B. Each layer is cheaper and coarser than the one above it, so failing fast at a cheap layer avoids wasting compute on expensive checks
   - C. Semantic checks cannot run unless a checksum has already passed
   - D. The order has no effect on cost or outcome, it is purely a style convention

3. What does Layer 4 (row-level hash) reveal that Layer 3 (checksum) cannot?
   - A. Whether the row counts match
   - B. Which specific rows diverge, rather than just whether some divergence exists somewhere in the table
   - C. Whether the business logic classifications are correct
   - D. The total dollar variance across the whole table

4. In the `ROW_NUMBER` bonus-tier bug, why did the discrepancy pass both row-count and sum-of-revenue reconciliation cleanly?
   - A. The bug only affected rows with tied revenue values, which reassigned rank without changing total row count or total revenue
   - B. The reconciliation job was misconfigured and skipped those checks
   - C. Oracle and Spark computed different total revenue, but the difference was too small to detect
   - D. The bug deleted and re-inserted rows with the same count

5. What is the correct fix for a non-deterministic `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY revenue DESC)`?
   - A. Remove the `PARTITION BY` clause
   - B. Switch from `ROW_NUMBER()` to `RANK()`
   - C. Append a unique column as a secondary sort key so no two rows can ever compare as equal
   - D. Run the query on a single-node cluster to force deterministic ordering

6. Why does summing a money column typed as `DOUBLE` produce drift that Oracle's `NUMBER`-based sum did not?
   - A. `DOUBLE` has a lower maximum value than `NUMBER`
   - B. `DOUBLE` is IEEE 754 binary floating point, which cannot represent most decimal fractions exactly, and that per-row error compounds across a large sum
   - C. Databricks does not support `SUM()` on `DOUBLE` columns
   - D. `NUMBER` and `DOUBLE` sort in different directions by default

7. What is the correct way to migrate an Oracle `NUMBER(12, 2)` column used for money?
   - A. Map it to Databricks `DOUBLE` for maximum compatibility
   - B. Map it to Databricks `DECIMAL(12, 2)`, matching precision and scale exactly
   - C. Map it to a bare `DECIMAL` with no precision or scale specified
   - D. Map it to `FLOAT` since Spark optimizes floating-point arithmetic

8. In a tiered reconciliation tolerance policy, which layer is the primary candidate for a nonzero tolerance threshold, and why?
   - A. Count, because row totals naturally fluctuate between systems
   - B. Sum, because correctly typed `DECIMAL` columns can still carry a few fractions of a cent of legitimate rounding-mode variance between engines at scale
   - C. Hash, because row-level fingerprints are expected to differ slightly
   - D. Semantic, because business-rule classifications are inherently approximate

9. Why should a `ROW_NUMBER` tie-break ordering divergence never be absorbed by a tolerance threshold, unlike sum-layer rounding drift?
   - A. It is a discrete logical misassignment (a wrong rank), not a small continuous numeric variance a threshold formula can bound
   - B. Tolerance thresholds only apply to `DECIMAL` columns
   - C. `ROW_NUMBER` results are always identical across engines, so no threshold is needed
   - D. It would require a negative tolerance value, which is not supported in SQL

10. What is the risk of setting a reconciliation tolerance threshold specifically to make a currently failing check pass?
    - A. There is no risk; thresholds exist precisely to be tuned until checks pass
    - B. It masks a real discrepancy behind a threshold chosen for convenience rather than derived from expected legitimate variance, undermining the reconciliation gate's purpose
    - C. It will cause the reconciliation job to fail to execute
    - D. Databricks enforces a fixed minimum tolerance that cannot be overridden

## Answer Key

1. **B** -- Identical row counts say nothing about whether individual rows, columns, or aggregates carry the correct values; a table can be fully wrong and still count-match.
2. **B** -- Running cheap, coarse checks first (count, sum) fails fast on obvious problems before spending compute on expensive row-level hash or semantic comparisons.
3. **B** -- Checksum confirms a divergence exists somewhere in a column; hash localizes it to specific rows via a join-key comparison (e.g. `EXCEPT`).
4. **A** -- The bug only reassigned rank among rows with tied revenue values -- it never changed which rows existed or what the total revenue was, so count and sum both passed clean.
5. **C** -- Adding a unique secondary sort key (e.g. a primary key column) guarantees every row has a fully deterministic position regardless of engine or execution plan.
6. **B** -- `DOUBLE`'s binary floating-point representation cannot exactly store most base-10 fractions, so tiny per-row errors accumulate into a real, visible drift across a large `SUM()`.
7. **B** -- Matching the Oracle column's exact precision and scale in a Databricks `DECIMAL(p, s)` preserves fixed-point, exact arithmetic equivalent to Oracle's `NUMBER(p, s)`.
8. **B** -- Sum is the layer where genuine, bounded rounding-mode variance between correctly typed `DECIMAL` arithmetic on two different engines is expected and should be tolerated within a derived threshold.
9. **A** -- A tie-break bug produces a wrong discrete outcome (row assigned the wrong rank), not a small numeric drift that a percentage-based tolerance formula is designed to bound.
10. **B** -- Tuning a threshold reactively to silence a specific failure defeats the reconciliation gate's purpose; thresholds must be derived from expected legitimate variance and documented, not adjusted to clear an inconvenient alert.

<!-- prevnext:start -->

---

| [&larr; Previous: Tolerance Thresholds: Acceptable vs Real Variance]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/tolerance-thresholds-acceptable-vs-real-variance/' | relative_url }}) | [Next: Building the Reconciliation Engine &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

