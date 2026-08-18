---
title: "Check Your Knowledge"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 7
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of this section's pattern translations before moving on.

1. Why is a cursor loop translated as `for row in df.collect(): ...` considered an anti-pattern?
   - A. `collect()` is deprecated in current Spark versions
   - B. It pulls the entire DataFrame to the driver, discarding distributed execution and risking out-of-memory failures
   - C. Python for-loops cannot iterate over Spark DataFrames at all
   - D. It requires a paid Databricks license feature

2. Which cursor pattern maps directly to a window function like `SUM(...) OVER (ORDER BY ...)`?
   - A. A per-row external API call
   - B. A running accumulation or running total maintained across rows in order
   - C. A cursor that never reads any rows
   - D. A cursor used only for logging

3. What replaces an Oracle row-level trigger on Databricks, since Delta tables don't execute code on write?
   - A. A second trigger defined on the same table
   - B. Delta Change Data Feed, consumed explicitly via `table_changes()` or `readChangeFeed`
   - C. A stored procedure called manually after every write
   - D. There is no replacement; triggers cannot be migrated

4. What is the main advantage of Lakeflow Jobs over `DBMS_SCHEDULER` highlighted in this section?
   - A. Lakeflow Jobs run faster in all cases
   - B. Scheduling, retries, and dependencies are declared visibly and every run produces a queryable history
   - C. Lakeflow Jobs don't require any configuration
   - D. DBMS_SCHEDULER cannot run PL/SQL code

5. When should a session temp table be translated into a CTE rather than a persisted Declarative Pipeline table?
   - A. Always -- CTEs should replace every temp table
   - B. When the intermediate result is used once, immediately, and never inspected independently
   - C. When the intermediate result is shared across many downstream steps
   - D. Never -- CTEs are not supported on Databricks

6. What does Delta's `MERGE INTO` do differently from Oracle's `MERGE` when the join condition matches more than one source row to a target row?
   - A. It silently picks the first matching row
   - B. It throws an error rather than applying a nondeterministic update
   - C. It ignores the match entirely
   - D. It automatically deduplicates the source table

7. Which clause does Delta's `MERGE INTO` support that Oracle's `MERGE` does not have a direct equivalent for?
   - A. `WHEN MATCHED`
   - B. `WHEN NOT MATCHED`
   - C. `WHEN NOT MATCHED BY SOURCE`
   - D. `USING`

8. What parameter does `create_auto_cdc_flow` use to correctly handle late-arriving or out-of-order change records in an SCD Type 2 flow?
   - A. `keys`
   - B. `target`
   - C. `sequence_by`
   - D. `stored_as_scd_type`

9. Why is unconditionally materializing every temp table as its own persisted Declarative Pipeline table listed as an anti-pattern?
   - A. Declarative Pipelines don't support persisted tables
   - B. It inflates storage and pipeline DAG complexity with nodes that add no value for single-use staging
   - C. It's illegal under Unity Catalog governance rules
   - D. It causes MERGE statements to fail

10. What do all five anti-patterns in the gallery have in common?
    - A. They all involve syntax errors that fail immediately
    - B. They preserve the legacy code's literal shape instead of its intent, and fail only at production scale or under real-world conditions
    - C. They only affect tables smaller than one gigabyte
    - D. They are all related to Unity Catalog permissions

## Answer Key

1. **B** -- `collect()` brings all rows to the driver, defeating distributed execution and risking out-of-memory errors at real data volume.
2. **B** -- A running total or accumulation walked in row order is exactly what a window function computes without any loop.
3. **B** -- Delta tables don't execute code on write, so Change Data Feed makes row-level changes queryable explicitly instead.
4. **B** -- Lakeflow Jobs make scheduling, retries, and dependencies visible in the job definition and produce a queryable run history.
5. **B** -- A single-use, immediately-consumed intermediate result doesn't need persisted storage; a CTE computes it inline.
6. **B** -- Delta's `MERGE INTO` rejects ambiguous matches with an error instead of silently applying a nondeterministic update.
7. **C** -- `WHEN NOT MATCHED BY SOURCE` handles target rows with no matching source row, a clause Oracle's `MERGE` lacks directly.
8. **C** -- `sequence_by` tells the AUTO CDC flow which column determines correct ordering, handling out-of-order arrivals correctly.
9. **B** -- Materializing single-use staging as a persisted table adds storage and DAG nodes that provide no benefit over a CTE.
10. **B** -- Each anti-pattern compiles and runs but preserves the legacy shape rather than the intent, failing only at production scale or under real arrival conditions.

<!-- prevnext:start -->

---

| [&larr; Previous: Anti-Pattern Gallery and Pattern Cheat Sheet]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/anti-pattern-gallery-and-pattern-cheat-sheet/' | relative_url }}) | [Next: AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

