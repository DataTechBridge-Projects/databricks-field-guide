---
title: "Check Your Knowledge"
parent: "Physical Design for Delta: Liquid Clustering Over Indexes"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of this section's physical design concepts before moving on.

1. What mechanism does Delta Lake use in place of a dedicated index structure to avoid scanning irrelevant data?
   - A. Bitmap indexes rebuilt nightly
   - B. Data skipping, driven by min/max column statistics per file
   - C. In-memory row caching
   - D. A separate index table joined at query time

2. Compared to an Oracle B-tree index, when is Liquid Clustering's reorganization work applied?
   - A. Synchronously on every row insert or update
   - B. Only when a DBA manually triggers a rebuild
   - C. Lazily during writes and background `OPTIMIZE` maintenance
   - D. Once, at table creation time, and never again

3. What command changes a table's clustering strategy without a full rewrite of existing data?
   - A. `DROP INDEX` followed by `CREATE INDEX`
   - B. `ALTER TABLE ... CLUSTER BY (...)`
   - C. `TRUNCATE TABLE` followed by reload
   - D. `OPTIMIZE ... ZORDER BY`

4. A table was previously `PARTITIONED BY (region)` with periodic `OPTIMIZE ... ZORDER BY (customer_id)`. What's the recommended starting point for its Liquid Clustering keys?
   - A. Start from scratch with a fresh workload analysis, ignoring the old definition
   - B. Combine the partition and Z-Order columns: `(region, customer_id)`
   - C. Use only the partition column and discard the Z-Order column
   - D. Cluster by the table's primary key instead

5. Which of the following is a documented limitation on Liquid Clustering keys?
   - A. Only one clustering key is allowed per table
   - B. Clustering keys must be numeric types only
   - C. Up to four clustering keys are supported, and `MAP`/`ARRAY` types are not
   - D. Clustering keys must match the table's partition columns exactly

6. When is `CLUSTER BY AUTO` the better choice over manually pinned clustering keys?
   - A. When the query pattern is genuinely unknown or still evolving
   - B. When the table is a small, rarely-queried dimension table
   - C. When the table has fewer than four columns total
   - D. Never -- manual keys always outperform automatic selection

7. According to the anti-pattern lecture, why doesn't over-clustering every table add write-path cost the way an extra Oracle index would?
   - A. Because Liquid Clustering never actually reorganizes any files
   - B. Because clustering is applied lazily, so the cost shows up as wasted background optimization compute instead of insert-time latency
   - C. Because Databricks charges a flat monthly fee regardless of clustering
   - D. Because clustered tables are automatically cached in memory

8. Which of these tables is the anti-pattern lecture's best candidate for staying unclustered?
   - A. A large fact table filtered by `order_date` on every dashboard query
   - B. A small dimension table with a few thousand rows that already fits in one or two files
   - C. A table with a well-understood, stable, high-selectivity query pattern
   - D. A table migrated from a heavily indexed Oracle schema

9. On the physical design decision card, what should you do if two candidate clustering keys are highly correlated (e.g. `order_date` and `order_month`)?
   - A. Include both to maximize data skipping
   - B. Drop the coarser-grained column and keep the finer-grained one
   - C. Drop both and cluster by a different column instead
   - D. Alternate between them on different `OPTIMIZE` runs

10. Where should the physical design decision card be applied for best results?
    - A. Only after production users file slow-query complaints
    - B. Before writing the table's `CREATE TABLE` statement, using data already gathered during the workload inventory
    - C. Only on tables larger than one terabyte
    - D. Once per migration project, applied to a single representative table

## Answer Key

1. **B** -- Delta Lake tracks min/max statistics per column per file and uses them to skip files that can't match a query's filter, rather than maintaining a separate index object.
2. **C** -- Clustering reorganization happens lazily during writes and background `OPTIMIZE` runs, not synchronously on every row like a B-tree index update.
3. **B** -- `ALTER TABLE ... CLUSTER BY (...)` changes the clustering strategy in place; existing data doesn't need to be rewritten to adopt it.
4. **B** -- Databricks' migration guidance is to combine existing partition and Z-Order columns as a like-for-like starting point, refined later from workload analysis.
5. **C** -- Up to four clustering keys are supported, and the supported types exclude `MAP` and `ARRAY`.
6. **A** -- `CLUSTER BY AUTO` fits tables whose query pattern hasn't stabilized yet, letting Databricks choose and adjust keys from observed traffic.
7. **B** -- The lazy application of clustering moves the cost from write-path latency to background optimization compute, which is wasted if the table's access pattern doesn't benefit.
8. **B** -- A small table that already fits in one or two files gains nothing from data skipping, since there's little to skip.
9. **B** -- Keeping both correlated columns wastes a limited key slot on redundant information; the finer-grained column already covers the coarser one's use cases.
10. **B** -- Applying the card before table creation, using data already collected during the workload inventory, avoids retrofitting clustering after performance complaints arrive.

<!-- prevnext:start -->

---

| [&larr; Previous: The Physical Design Decision Card]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/the-physical-design-decision-card/' | relative_url }}) | [Next: The Procedure Autopsy: Decomposing PL/SQL &rarr;]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

