---
title: "Liquid Clustering vs Z-Ordering vs Partitioning"
parent: "Physical Design for Delta: Liquid Clustering Over Indexes"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/liquid-clustering-vs-z-ordering-vs-partitioning/
read_minutes: 2
---

# Liquid Clustering vs Z-Ordering vs Partitioning
{: .no_toc }

*Estimated read: 2 min*

Delta Lake actually has three physical layout techniques with overlapping goals, and a migration
that just landed on the platform will often find older tables or tutorials still using the first
two. Knowing what each one does -- and that Liquid Clustering is the one to reach for by default now
-- avoids re-implementing a legacy indexing habit under a new name.

| | **Partitioning** | **Z-Ordering** | **Liquid Clustering** |
|---|---|---|---|
| **How it works** | Physically splits data into separate directories by column value (e.g. one folder per `order_date`) | Co-locates rows with similar values across multiple columns within files, applied via `OPTIMIZE ... ZORDER BY` | Continuously, incrementally reorganizes files so rows with similar clustering-key values stay co-located |
| **Changing the strategy** | Requires a full table rewrite to repartition | Requires re-running `OPTIMIZE ZORDER BY` over the whole table | `ALTER TABLE ... CLUSTER BY (...)` -- no rewrite of existing data required |
| **Small-file problem** | Common with high-cardinality partition columns (e.g. partitioning by `customer_id`) -- each partition can end up with tiny files | Doesn't create the partition explosion problem, but only clusters data written since the last `OPTIMIZE` run | Handled automatically as part of ongoing background optimization |
| **Best fit** | Legacy pattern; still useful for genuinely low-cardinality columns with a hard business meaning (e.g. multi-tenant `region`) | Superseded by Liquid Clustering for new tables | Default recommendation for new Delta tables |

**Migrating an existing partitioned or Z-Ordered table to Liquid Clustering doesn't require
choosing new columns from scratch.** Databricks' own migration guidance is direct: if a table is
currently partitioned, use those same partition columns as your clustering keys; if it's
Z-Ordered, use the `ZORDER BY` columns; if it's both, combine the two column sets. That gives you a
like-for-like starting point you can refine later based on the query-pattern analysis in the next
lecture, rather than re-deriving clustering keys from nothing.

```sql
-- A table that was previously: PARTITIONED BY (region) with periodic
-- OPTIMIZE ... ZORDER BY (customer_id)
ALTER TABLE orders CLUSTER BY (region, customer_id);
```

{: .important }
> Don't partition a new Delta table by a high-cardinality column out of legacy habit -- the
> directory-per-value explosion that made wide partitioning painful in Hive-style tables is still
> painful in Delta. Liquid Clustering exists specifically to avoid that tradeoff: it gets
> partitioning's data-skipping benefit without forcing a physical directory boundary per value.

One more reason to default to Liquid Clustering on new tables: Databricks also supports
`CLUSTER BY AUTO` on Unity Catalog–managed tables, which lets the platform choose and continuously
adjust clustering keys based on observed query patterns instead of requiring you to declare them
upfront -- covered alongside manual key selection in the next lecture.

<!-- prevnext:start -->

---

| [&larr; Previous: Oracle Has Muscles, Delta Has a Nervous System]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/oracle-has-muscles-delta-has-a-nervous-system/' | relative_url }}) | [Next: Choosing Cluster Keys from Query Patterns &rarr;]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/choosing-cluster-keys-from-query-patterns/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

