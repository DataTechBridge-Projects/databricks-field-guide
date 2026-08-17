---
title: "Check Your Knowledge"
parent: "Lakeflow Spark Declarative Pipelines - Transformation Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- SDP's declarative model, destinations, flows,
AUTO CDC, and production patterns.

1. What is the key difference between imperative pipeline code (Sections 5-8) and SDP's declarative model?
   A. Declarative code cannot use Python
   B. You describe each destination table's logic and SDP infers dependency order and manages checkpoints automatically
   C. Declarative pipelines cannot process streaming data
   D. There is no real difference

2. What is the difference between a streaming table and a materialized view as SDP destinations?
   A. They are identical
   B. A streaming table is incrementally processed for continuously arriving data; a materialized view is (re)computed based on current full state, right for aggregations
   C. Materialized views cannot be queried with SQL
   D. Streaming tables are only for gold-layer data

3. What does the `sequence_by` argument in an AUTO CDC flow determine?
   A. The order columns appear in the output table
   B. Which incoming change is the true latest version, when multiple changes for the same key arrive
   C. The partition key for the destination table
   D. How often the pipeline runs

4. What is the structural difference between SCD Type 1 and SCD Type 2 in AUTO CDC?
   A. Type 1 is faster but functionally identical to Type 2
   B. Type 1 overwrites in place, keeping only current state; Type 2 preserves full history via __START_AT/__END_AT
   C. Type 2 is only available in SQL, not Python
   D. Type 1 requires a snapshot source; Type 2 requires a CDC feed

5. What does `expect_or_fail` do when a row violates its condition?
   A. Nothing -- it behaves like plain `expect`
   B. The violating row is silently dropped
   C. The entire pipeline run fails
   D. The row is automatically corrected

6. Why is `AUTO CDC FROM SNAPSHOT` used instead of standard AUTO CDC?
   A. It is always faster
   B. It handles sources that only provide periodic full snapshots, with no row-level change feed available
   C. It only works with SQL warehouses
   D. It replaces the need for keys

7. According to this section's production guidance, why should `expect_or_fail` be reserved for genuinely non-negotiable invariants rather than used everywhere by default?
   A. It is not supported in production pipelines
   B. Overly strict failure conditions on minor issues train people to ignore or disable pipeline failures
   C. It only works on materialized views
   D. It disables the event log

8. Why should a large project generally be split into multiple SDP pipelines rather than one giant pipeline?
   A. Pipelines cannot contain more than five tables
   B. A single massive pipeline makes the dependency graph unreadable and couples unrelated tables' failure/monitoring together
   C. Multiple pipelines run faster automatically
   D. Unity Catalog requires one pipeline per catalog

9. How can SDP transformation logic be unit tested?
   A. It cannot be tested outside a running pipeline
   B. By extracting the core logic into a plain Python function, called from the `@dp.table`-decorated function, and testing that plain function directly
   C. Only by running the full pipeline in production
   D. Testing is handled automatically by SDP

10. What determines whether SDP performs a full or incremental refresh of a materialized view?
    A. A manual configuration flag you must always set explicitly
    B. Automatically, based on whether the query's structure allows computing only the delta from new data
    C. It is always a full refresh
    D. It depends solely on the destination catalog

## Answer Key

1. **B** -- SDP infers dependency order and manages checkpoints from declared table logic, rather than requiring manual orchestration code.
2. **B** -- streaming tables suit incremental data; materialized views suit aggregations computed from current state.
3. **B** -- `sequence_by` establishes true change order among incoming changes for the same key.
4. **B** -- Type 1 overwrites in place; Type 2 preserves full history with start/end columns.
5. **C** -- `expect_or_fail` fails the entire pipeline run on violation.
6. **B** -- it's for sources without a CDC feed, comparing snapshots instead.
7. **B** -- overuse of hard failures on minor issues erodes trust in pipeline alerts over time.
8. **B** -- splitting pipelines keeps dependency graphs readable and isolates unrelated failures.
9. **B** -- extracting plain functions from the decorated table function allows standard unit testing.
10. **B** -- SDP automatically determines refresh strategy based on whether the query structure permits an incremental delta computation.

<!-- prevnext:start -->

---

| [&larr; Previous: SDP Design Decisions and Production Patterns]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/sdp-design-decisions-and-production-patterns/' | relative_url }}) | [Next: Lakeflow Jobs - The Orchestration Pillar &rarr;]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

