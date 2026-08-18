---
title: "Check Your Knowledge"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of building a production reconciliation engine before moving on to the parallel run and cutover.

1. In the reconciliation job's three parity levels, what is the purpose of running count parity before hash parity?
   A. Count parity is more accurate than hash parity
   B. It's a cheap gate that aborts the job early, avoiding wasted compute on hash checks when rows are already known to be missing or duplicated
   C. Hash parity cannot run without a prior count check
   D. Count parity replaces the need for a numeric tolerance level

2. Why should the reconciliation audit table be append-only rather than overwritten on each run?
   A. Delta Lake does not support overwriting tables
   B. Append-only tables use less storage
   C. It preserves a permanent, queryable history of every parity check ever run, which is the artifact that resolves later audits or disputes
   D. Overwriting causes schema evolution errors

3. In the PySpark hash-diff function, why is `coalesce()` used before `concat_ws()` when building the row hash?
   A. To improve join performance
   B. `concat_ws` silently drops NULL arguments, so without a sentinel value, a NULL on one side and an empty string on the other can hash identically and hide a real mismatch
   C. To convert all columns to the same data type
   D. `coalesce` is required syntax for `sha2()`

4. What does the `diff_type` column (missing_in_source, missing_in_target, value_mismatch) add beyond a simple hash inequality check?
   A. It improves the hash function's collision resistance
   B. It removes the need for a full outer join
   C. It distinguishes rows missing entirely on one side from rows present on both sides with different values, pointing an engineer toward ingestion versus transformation as the likely cause
   D. It is required for Delta Lake's schema enforcement

5. On a reconciliation dashboard, what question does the status heat map answer that the big open-mismatch counter cannot?
   A. Whether the migration is on schedule
   B. Which specific table pairs and run dates are failing, localizing a problem the aggregate counter can only signal exists
   C. The total dollar value of the migration
   D. Whether the source system is Oracle or SQL Server

6. Why should a reconciliation dashboard be built against the Delta audit table rather than recomputing diffs live at view time?
   A. Live queries are not supported in Databricks SQL
   B. The audit table is cheaper to store than to query
   C. A live recompute is slow, expensive, and gives every viewer a different answer depending on when they load the page, whereas the audit table is a fixed, shared record
   D. Delta tables cannot be queried by dashboards

7. What is the core argument for running reconciliation nightly starting on day zero of a migration, rather than only in the weeks before cutover?
   A. Nightly jobs are required by Lakeflow Jobs
   B. It follows the defect cost curve -- a bug caught the day it's introduced is far cheaper to fix than the same bug discovered weeks later against a much larger, less familiar codebase
   C. It reduces the total number of tables that need to be reconciled
   D. Databricks clusters are cheaper to run at night

8. Beyond catching data bugs sooner, what additional benefit does running the reconciliation engine nightly from day zero provide?
   A. It eliminates the need for a hash-diff implementation
   B. It removes the need for a tolerance threshold
   C. It validates the reconciliation engine itself against real data early, while there's still time to fix a broken hash function or join key before deadline pressure hits
   D. It automatically resolves all detected mismatches

9. In the reconciliation script library, what is the role of the per-table-pair YAML config file?
   A. It replaces the need for a join key
   B. It defines source and target connections, the join key, compared columns, and tolerance for one table pair, so a single engine can be reused across every pair without new code
   C. It stores the actual row-level diff output
   D. It is only used for the dashboard, not the reconciliation job itself

10. Why does the lecture recommend extending the YAML config schema for a "difficult" table pair rather than writing it a bespoke script?
    A. YAML files run faster than Python scripts
    B. Bespoke scripts are not supported by Lakeflow Jobs
    C. A second, table-specific code path is easy to forget to keep in sync when the shared engine's logic changes, undermining the whole point of a single reusable engine
    D. Bespoke scripts cannot write to Delta tables

## Answer Key

1. **B** -- Count parity is the cheapest check and runs first specifically to abort before more expensive hash computation on data already known to be wrong.
2. **C** -- The audit table's value as a trust ledger depends on it never losing a prior run's record.
3. **B** -- `concat_ws` drops NULLs silently, so an explicit sentinel is required to make NULL-vs-empty-string a detectable difference.
4. **C** -- Classifying the diff type points the investigation toward the right stage of the pipeline (ingestion vs. transformation) instead of just flagging "mismatch."
5. **B** -- The heat map's two axes (table pair, run date) localize exactly where a divergence is occurring, which a single counter cannot show.
6. **C** -- Recomputing hashes at view time is expensive and non-deterministic across viewers; the audit table is the fixed source of truth.
7. **B** -- The defect cost curve is the core rationale: catching bugs early is dramatically cheaper than catching them late.
8. **C** -- Early, continuous use of the engine on real data is also how the reconciliation tooling itself gets battle-tested well before cutover pressure arrives.
9. **B** -- The YAML config is what lets one engine drive reconciliation for any table pair without table-specific code.
10. **C** -- A special-cased script creates a second logic path that silently drifts out of sync with fixes made to the shared engine.

<!-- prevnext:start -->

---

| [&larr; Previous: The Reconciliation Script Library]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-reconciliation-script-library/' | relative_url }}) | [Next: The Parallel Run and Zero-Downtime Cutover &rarr;]({{ '/03-legacy-migration-to-databricks/14-the-parallel-run-and-zero-downtime-cutover/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

