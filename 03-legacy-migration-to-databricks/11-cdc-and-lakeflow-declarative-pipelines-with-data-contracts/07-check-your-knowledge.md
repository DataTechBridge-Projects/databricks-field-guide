---
title: "Check Your Knowledge"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 7
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of CDC architecture, Auto Loader schema evolution, `AUTO CDC` for SCD Type 2, and data contracts before moving on.

1. In the Databricks-native CDC chain, what plays the role that a hand-written `MERGE INTO` plus effective-dating logic played in a legacy warehouse?
   - A. Auto Loader
   - B. `AUTO CDC` inside a Lakeflow Declarative Pipeline
   - C. Unity Catalog
   - D. `OPTIMIZE`

2. By default, how does Auto Loader infer types for loosely typed formats like JSON and CSV?
   - A. It always infers the most specific type it can detect
   - B. It defaults every column to `STRING` unless overridden with schema hints
   - C. It refuses to infer a schema and requires one to be supplied manually
   - D. It infers types only from the first row of the first file

3. Which `cloudFiles.schemaEvolutionMode` setting causes a new, unrecognized column to be silently dropped with no failure and no rescue?
   - A. `addNewColumns`
   - B. `failOnNewColumns`
   - C. `rescue`
   - D. `none`

4. Under the default `addNewColumns` evolution mode, what happens the first time Auto Loader encounters a genuinely new column?
   - A. The new column is silently dropped
   - B. The stream fails once, the schema is updated, and a restart resumes processing with the new column included
   - C. The entire pipeline is rolled back to its last checkpoint
   - D. The new column is routed to a `_rescued_data` column and processing continues uninterrupted

5. In an `AUTO CDC` flow configured with `stored_as_scd_type="2"`, what do the generated `__START_AT` and `__END_AT` columns represent?
   - A. The first and last time the pipeline itself ran
   - B. The validity window of each historical version of a row, with `__END_AT IS NULL` marking the current version
   - C. The retry window Databricks uses before failing a flow
   - D. The schema inference sample window Auto Loader used

6. Why does `sequence_by` matter to `AUTO CDC`'s correctness?
   - A. It determines the cluster size used to process the flow
   - B. It ensures changes are applied in the correct logical order even if they arrive out of order due to network retries or partition skew
   - C. It sets the retention period for historical SCD Type 2 versions
   - D. It is required only for `SCD TYPE 1`, not `SCD TYPE 2`

7. What is the key difference between `expect_or_drop` and `expect_or_fail` as data-contract enforcement operators?
   - A. `expect_or_drop` halts the pipeline; `expect_or_fail` only logs a warning
   - B. `expect_or_drop` removes the failing row and keeps the pipeline running; `expect_or_fail` stops the pipeline and rolls back the update
   - C. They behave identically but apply to different table types
   - D. `expect_or_fail` can only be used on gold-layer tables

8. In the schema-drift incident where `cloudFiles.schemaEvolutionMode` was set to `none` and no contract enforced `NOT NULL` on the business key, why did the silver dimension accumulate phantom SCD Type 2 versions instead of simply missing data?
   - A. `AUTO CDC` retried every failed row automatically, each retry creating a new version
   - B. Every incoming row had a `NULL` business key, and since `NULL` never equals `NULL` in the merge-key comparison, each row was treated as a new, unmatched key rather than an update
   - C. The gold layer aggregated the same customer multiple times due to a `JOIN` error
   - D. Auto Loader re-ingested the same files repeatedly

9. What is the core architectural problem with the direct-to-silver anti-pattern (pointing `AUTO CDC` straight at a raw source stream with no bronze table)?
   - A. It uses more storage than a layered design
   - B. It removes the durable, replayable checkpoint and the natural place to attach a data contract, collapsing ingestion and transformation into one failure domain
   - C. It is not supported by Lakeflow Declarative Pipelines and will fail to deploy
   - D. It prevents `AUTO CDC` from supporting `SCD TYPE 2`

10. In the bronze contract template, which three conditions are treated as non-negotiable `expect_or_fail` checks for virtually any CDC source?
    - A. Order total non-negative, status code in a fixed enum, and email format valid
    - B. Business key not null, operation code recognized, and sequencing column not null
    - C. File size under a threshold, partition count fixed, and cluster size sufficient
    - D. Schema hints present, checkpoint location set, and trigger interval configured

## Answer Key

1. **B** -- `AUTO CDC` (formerly `APPLY CHANGES`) inside a Lakeflow Declarative Pipeline replaces the hand-written `MERGE INTO` and effective-dating logic a legacy team maintained manually.
2. **B** -- Auto Loader defaults loosely typed formats to `STRING` for every column, deliberately conservative unless overridden with `cloudFiles.schemaHints`.
3. **D** -- `none` mode ignores new columns entirely: no failure, no rescue, and the column is silently dropped.
4. **B** -- The default `addNewColumns` mode fails the stream once when it sees a new column, then resumes with the updated schema on restart.
5. **B** -- `__START_AT` and `__END_AT` mark each historical version's validity window; a `NULL` `__END_AT` identifies the current version.
6. **B** -- `sequence_by` tells `AUTO CDC` which column determines logical ordering, so out-of-order delivery doesn't apply an older change over a newer one.
7. **B** -- `expect_or_drop` quarantines the failing row and keeps running; `expect_or_fail` stops the pipeline and rolls back the update entirely.
8. **B** -- With the business key dropped to `NULL` for every row, `AUTO CDC`'s merge-key comparison treated every row as unique (since `NULL` never equals `NULL`), opening a new phantom version on each micro-batch instead of updating existing rows.
9. **B** -- Skipping bronze removes the replayable raw copy and the natural attachment point for a contract, so ingestion and transformation failures become indistinguishable and unrecoverable from within Databricks alone.
10. **B** -- A non-null business key, a recognized operation code, and a non-null sequencing column are the three structural checks `AUTO CDC` depends on regardless of source; everything else is source-specific.

<!-- prevnext:start -->

---

| [&larr; Previous: Anti-Pattern: Direct-to-Silver and the Contract Template]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/anti-pattern-direct-to-silver-and-the-contract-template/' | relative_url }}) | [Next: The Reconciliation Stack: Proving Semantic Parity &rarr;]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

