---
title: "Check Your Knowledge"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of this section's ingestion decisions before moving on.

1. In the "data ferry" framing, what does the **manifest** term of the ingestion contract enforce?
   - A. How often the pipeline runs
   - B. How much compute the cluster is allowed to use
   - C. That the expected files or rows actually arrived, rather than silently assuming a partial load is complete
   - D. Which file format the source system exports

2. Why does a fail-fast manifest check matter more than porting a legacy job's happy path alone?
   - A. It makes the pipeline run faster
   - B. Without it, a missing upstream file can produce a silently short result instead of a loud failure
   - C. Databricks requires a manifest check to start any job
   - D. It replaces the need for a schedule entirely

3. What determines whether an ingestion workload should be batch or streaming?
   - A. The size of the source table
   - B. The latency requirement of the downstream consumer, not technical sophistication
   - C. Whether the source is on-premises or in the cloud
   - D. The file format being ingested

4. What does `Trigger.AvailableNow` let a Structured Streaming job do?
   - A. Run continuously forever with no stopping condition
   - B. Process available data like a batch job while still using streaming's checkpoint-tracked, exactly-once machinery
   - C. Skip checkpoint management entirely
   - D. Convert a streaming job into a JDBC bulk pull

5. What is the main practical driver behind choosing push vs. pull ingestion?
   - A. The Databricks Runtime version
   - B. Whether the migration team controls and can directly query the source system
   - C. The number of columns in the source table
   - D. Whether Unity Catalog is enabled

6. Where does schema-on-read fit best in the medallion architecture, and why?
   - A. In gold, because reports need the most flexible schema
   - B. In bronze, so data can be captured even if the source schema drifts, with enforcement applied later
   - C. Nowhere -- Databricks only supports schema-on-write
   - D. In silver, replacing schema-on-write entirely

7. At 10TB, why does a JDBC bulk pull with many parallel partitions become a poor default?
   - A. JDBC cannot read tables larger than 1TB
   - B. Parallel range-scan queries compete directly with production OLTP traffic on the source database
   - C. Auto Loader is always faster regardless of source
   - D. JDBC does not support the `partitionColumn` option

8. What does Auto Loader track to guarantee exactly-once, incremental file processing?
   - A. A manually maintained watermark column
   - B. RocksDB-backed checkpoint state
   - C. The JDBC connection pool
   - D. A `.done` marker file per table

9. Why might a migration team choose a partner CDC tool (Fivetran, Qlik, Arcion) over Auto Loader for a source table?
   - A. Auto Loader cannot read Parquet files
   - B. The requirement is sub-minute latency on row-level updates and deletes captured from the source's transaction log
   - C. CDC tools are always cheaper than Auto Loader
   - D. Auto Loader requires a partner license to run

10. According to the ingestion pattern matrix, what should override the matrix's default recommendation for a given table?
    - A. Nothing -- the matrix's recommendation should always be followed exactly
    - B. A specific business requirement, such as reflecting deletes within one minute, that the default pattern can't satisfy
    - C. The alphabetical order of the source table name
    - D. Whichever tool was used for the previous table in the inventory

## Answer Key

1. **C** -- The manifest term confirms the expected files or rows actually arrived, catching a partial load before it's treated as complete.
2. **B** -- Skipping the fail-fast guard trades a loud, obvious failure for a silent one that surfaces later as a bad report.
3. **B** -- Batch vs. streaming is a latency-requirement question, not a measure of technical sophistication.
4. **B** -- `Trigger.AvailableNow` processes currently available data and stops, like a batch job, while reusing streaming's exactly-once checkpoint machinery.
5. **B** -- Push vs. pull mainly comes down to whether the migration team controls and can directly query the source system.
6. **B** -- Schema-on-read belongs in bronze, capturing drifting source data before enforcement is applied moving into silver.
7. **B** -- Many parallel JDBC range-scan queries place sustained load directly on the production source database.
8. **B** -- Auto Loader relies on RocksDB-backed checkpoint state to guarantee exactly-once, incremental processing.
9. **B** -- Sub-minute latency on row-level changes is exactly what log-based CDC tools deliver that Auto Loader's file-watching model doesn't.
10. **B** -- A concrete business requirement the default pattern can't meet should override the matrix's starting recommendation.

<!-- prevnext:start -->

---

| [&larr; Previous: The Ingestion Pattern Matrix]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/the-ingestion-pattern-matrix/' | relative_url }}) | [Next: CDC and Lakeflow Declarative Pipelines With Data Contracts &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

