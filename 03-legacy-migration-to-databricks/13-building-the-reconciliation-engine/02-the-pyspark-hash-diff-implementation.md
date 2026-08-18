---
title: "The PySpark Hash-Diff Implementation"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-pyspark-hash-diff-implementation/
read_minutes: 3
---

# The PySpark Hash-Diff Implementation
{: .no_toc }

*Estimated read: 3 min*

The SQL `EXCEPT` pattern from the prior section works fine when both tables already live in Databricks. In a real migration, the source is usually still Oracle or SQL Server, reachable through **Lakehouse Federation** or a landed extract, and the hash logic needs to run inside a PySpark job that can be scheduled, parameterized, and reused across dozens of table pairs -- not hand-typed SQL per table. This lecture builds that job.

The core function hashes every row on both sides the same way, using [`pyspark.sql.functions.sha2`](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.sha2.html) over a deterministic, delimiter-joined concatenation of the reconciled columns:

```python
from pyspark.sql import functions as F

def add_row_hash(df, cols, key_col):
    concat_expr = F.concat_ws(
        "|", *[F.coalesce(F.col(c).cast("string"), F.lit("<NULL>")) for c in cols]
    )
    return df.select(key_col, F.sha2(concat_expr, 256).alias("row_hash"))
```

Two details here are the ones that actually break naive implementations. First, **column order must be identical on both sides** -- `concat_ws` is positional, so if the source and target `cols` lists drift out of sync, every row hashes to a mismatch that has nothing to do with real data drift. Second, `NULL` needs an explicit sentinel via `coalesce`, because `concat_ws` silently drops `NULL` arguments rather than erroring, which means a row with a `NULL` in one system and an empty string in the other can hash identically and hide a real discrepancy.

With both sides hashed, a full outer join on the business key finds every category of mismatch in one pass:

```python
source_h = add_row_hash(source_df, recon_cols, "order_id")
target_h = add_row_hash(target_df, recon_cols, "order_id")

diff = (
    source_h.withColumnRenamed("row_hash", "source_hash")
    .join(
        target_h.withColumnRenamed("row_hash", "target_hash"),
        on="order_id",
        how="full_outer",
    )
    .withColumn(
        "diff_type",
        F.when(F.col("source_hash").isNull(), "missing_in_source")
        .when(F.col("target_hash").isNull(), "missing_in_target")
        .when(F.col("source_hash") != F.col("target_hash"), "value_mismatch")
        .otherwise("match"),
    )
    .filter(F.col("diff_type") != "match")
)
```

That `diff_type` column is the payoff: it separates rows that are genuinely missing on one side from rows that exist on both sides but disagree on content, which is exactly the distinction a migration engineer needs to decide whether the bug is in ingestion (dropped rows) or transformation (wrong values). Write `diff` straight into the Delta audit table from the previous lecture, and this function becomes the reusable engine the rest of the section builds a dashboard, a schedule, and a config library around.

A `full_outer` join scales fine to the row counts a table-by-table reconciliation typically deals with, but it's worth being deliberate about the join key's data type before running it at production volume. Casting `order_id` to a consistent type on both sides -- rather than relying on implicit coercion between an Oracle `NUMBER` key and a Spark `LongType` -- avoids a subtle failure mode where the join silently drops matches because two logically identical keys compare unequal after a mismatched cast. It's worth adding an assertion early in the job that both `source_h` and `target_h` have zero duplicate keys before the join runs, since a duplicated key on either side turns a clean one-to-one join into a fan-out that inflates the mismatch count with rows that were never actually wrong.

One more practical note: `sha2` with a 256-bit digest is the right default for this use case, wide enough that a collision between two genuinely different rows is not a realistic concern at any table size this reconciliation engine will see, and cheap enough to compute across millions of rows nightly without becoming the bottleneck in the job. Resist the temptation to hash fewer bits "for speed" -- the digest computation is rarely where a reconciliation job's runtime actually goes; the shuffle behind the full outer join is.

<!-- prevnext:start -->

---

| [&larr; Previous: Architecting the Reconciliation Job]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/architecting-the-reconciliation-job/' | relative_url }}) | [Next: Reconciliation Dashboards: Making Parity Visible &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/reconciliation-dashboards-making-parity-visible/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

