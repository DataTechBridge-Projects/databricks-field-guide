---
title: "Choosing Cluster Keys from Query Patterns"
parent: "Physical Design for Delta: Liquid Clustering Over Indexes"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/choosing-cluster-keys-from-query-patterns/
read_minutes: 3
---

# Choosing Cluster Keys from Query Patterns
{: .no_toc }

*Estimated read: 3 min*

Choosing clustering keys is the same exercise a DBA already runs when deciding which columns
deserve an index -- look at what queries actually filter, join, and group on -- adapted to Liquid
Clustering's specific rules and limits.

## Start from the workload inventory, not from the schema

The stored-procedure dependency graphs and query-log analysis from the Autopsy section (Oracle AWR
reports, Teradata DBQL, SQL Server Query Store) are the right input here, not a fresh guess. Pull
the columns that show up most often in:

- `WHERE` clause predicates, especially equality and range filters
- `JOIN` conditions, particularly on large fact tables joined to dimension tables
- `GROUP BY` clauses on high-volume aggregation queries

Rank them by how often they appear across the workload, not by how important they seem
individually -- a column referenced in one dashboard's edge-case filter matters less than one that
shows up in every nightly batch query against the table.

## The rules that shape the shortlist

Databricks' clustering-key guidance narrows that ranked list down to a workable choice:

- **Up to four clustering keys per table.** Unlike an index-per-query-pattern approach, you're
  choosing a small, shared set of columns that has to serve the table's *whole* query mix, not one
  structure per query.
- **Order keys from highest to lowest selectivity when cardinality is high.** For a large table, put
  the most selective, most-frequently-filtered column first.
- **Supported types** are `DATE`, `TIMESTAMP`, `STRING`, `INT`, `LONG`, `FLOAT`, `DOUBLE`,
  `DECIMAL`, and nested structs of those types -- not `MAP` or `ARRAY`.
- **Drop one of any two highly correlated columns.** If `order_date` and `order_month` are always in
  lockstep, clustering by both wastes a key slot on redundant information; keep the finer-grained
  one and let it cover both use cases.

```sql
ALTER TABLE orders
CLUSTER BY (customer_id, order_date);

-- Or let Databricks choose and continuously adjust keys based on observed
-- query patterns, on Unity Catalog-managed tables:
ALTER TABLE orders CLUSTER BY AUTO;
```

## When manual beats automatic

`CLUSTER BY AUTO` is the right default when a table's query pattern is genuinely unknown or still
evolving -- exactly the position most tables are in immediately after a migration, before real
production query traffic has accumulated against them. Once you have weeks of actual query-log
data, comparing what `AUTO` selected against what your own workload analysis would have chosen is a
useful sanity check, and manually pinning keys still makes sense for tables where you have
high-confidence knowledge that automatic selection can't yet infer -- a table you know will be
driven by a specific new reporting workload that hasn't gone live yet, for example.

{: .important }
> The single most common physical-design mistake carried over from a legacy migration is choosing
> clustering keys the way you'd choose index columns: one key per anticipated query, added
> incrementally over time. Liquid Clustering rewards the opposite instinct -- a small, deliberately
> chosen set of columns that serve the table's dominant access patterns, decided once from real
> workload data and revisited only when that workload materially changes.

<!-- prevnext:start -->

---

| [&larr; Previous: Liquid Clustering vs Z-Ordering vs Partitioning]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/liquid-clustering-vs-z-ordering-vs-partitioning/' | relative_url }}) | [Next: Anti-Pattern: Over-Clustering Every Table &rarr;]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/anti-pattern-over-clustering-every-table/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

