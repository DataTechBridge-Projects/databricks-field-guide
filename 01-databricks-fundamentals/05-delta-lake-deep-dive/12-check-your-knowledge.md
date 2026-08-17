---
title: "Check Your Knowledge"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 12
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- Delta Lake fundamentals, tables, time travel,
maintenance, `MERGE INTO`, schema evolution, and constraints.

1. What underlies Delta Lake's ACID guarantees on top of plain Parquet files?
   A. A proprietary binary file format
   B. The transaction log (`_delta_log`)
   C. A separate relational database
   D. Databricks' cluster manager

2. What happens when you `DROP TABLE` a **managed** Delta table?
   A. Only the catalog entry is removed; data files remain
   B. Both the catalog entry and the underlying data files are deleted
   C. Nothing happens until `VACUUM` runs
   D. The table becomes an external table automatically

3. Which SQL syntax queries a Delta table as it existed at a specific version number?
   A. `SELECT * FROM table SNAPSHOT 42`
   B. `SELECT * FROM table VERSION AS OF 42`
   C. `SELECT * FROM table WHERE version = 42`
   D. `ROLLBACK TABLE table TO 42`

4. What does `VACUUM` actually do?
   A. Compacts small files into larger ones
   B. Permanently deletes data files no longer referenced by the table and older than the retention threshold
   C. Restores a table to a prior version
   D. Validates schema constraints

5. What is the key structural difference between `RESTORE TABLE` and re-running a pipeline from source?
   A. `RESTORE` adds a new commit reverting to a prior state, in seconds, without needing source data
   B. `RESTORE` deletes all history between the restore point and now
   C. Re-running from source is always faster
   D. There is no difference

6. In a `MERGE INTO` statement, what does `WHEN NOT MATCHED BY SOURCE` handle?
   A. Rows present in both source and target
   B. Rows present in the target but no longer present in the source
   C. Rows present in the source but not yet in the target
   D. Duplicate rows in the source

7. Why is `MERGE INTO` generally idempotent when keyed correctly, while a plain `INSERT` often isn't?
   A. `MERGE` runs faster than `INSERT`
   B. Matched rows are updated (not duplicated) and unmatched rows insert exactly once on a rerun
   C. `INSERT` is deprecated in Delta Lake
   D. `MERGE` disables schema enforcement

8. What must you do before a write that adds a genuinely new column is allowed to succeed against a table with strict schema enforcement?
   A. Nothing -- new columns are always silently accepted
   B. Explicitly opt in with `mergeSchema`/`WITH SCHEMA EVOLUTION`, or run an explicit `ALTER TABLE`
   C. Drop and recreate the entire table
   D. Disable Unity Catalog

9. Are Delta Lake's `PRIMARY KEY` and `FOREIGN KEY` constraints enforced at write time?
   A. Yes, always
   B. No -- they are informational only and do not reject violating writes
   C. Only on Unity Catalog-managed tables
   D. Only for streaming writes

10. What is the `VARIANT` type primarily useful for?
    A. Replacing all `DECIMAL` columns for better precision
    B. Storing and directly querying semi-structured data without a rigid predefined schema
    C. A faster alternative to `BIGINT` for identity columns
    D. Enforcing `CHECK` constraints

## Answer Key

1. **B** -- the transaction log is what turns a folder of Parquet files into an ACID-compliant table.
2. **B** -- dropping a managed table removes both the metadata and the underlying data files.
3. **B** -- `VERSION AS OF` is the correct time travel syntax for a specific version number.
4. **B** -- `VACUUM` physically deletes unreferenced, expired data files to reclaim storage.
5. **A** -- `RESTORE` is a fast, forward-only commit that reverts the table's current state without needing to reprocess from source.
6. **B** -- `WHEN NOT MATCHED BY SOURCE` targets rows the source no longer has, useful for archiving/soft-deleting.
7. **B** -- `MERGE`'s match semantics naturally avoid duplication on rerun, unlike a naive append `INSERT`.
8. **B** -- schema evolution must be explicitly opted into per write, or applied via an explicit `ALTER TABLE`.
9. **B** -- primary/foreign keys in Delta Lake are informational only; they aid the optimizer but don't reject violations.
10. **B** -- `VARIANT` stores semi-structured data efficiently while remaining directly queryable without a fixed schema.

<!-- prevnext:start -->

---

| [&larr; Previous: Table constraints - NOT NULL, CHECK, and Identity Columns]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/table-constraints-not-null-check-and-identity-columns/' | relative_url }}) | [Next: Unity Catalog - Governance from day one &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

