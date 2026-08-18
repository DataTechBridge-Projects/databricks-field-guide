---
title: "Anti-Pattern: Over-Clustering Every Table"
parent: "Physical Design for Delta: Liquid Clustering Over Indexes"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/anti-pattern-over-clustering-every-table/
read_minutes: 3
---

# Anti-Pattern: Over-Clustering Every Table
{: .no_toc }

*Estimated read: 3 min*

The previous lecture's rules make Liquid Clustering easy to reach for on every table in a migrated
estate. That's exactly the instinct to resist. A DBA who spent a career adding indexes to fix slow
queries tends to bring the same reflex to Databricks: cluster everything, on every plausible column,
because it looks cheap. It isn't free, and the cost shows up in a different place than the old
indexing tradeoff did.

## Where the cost actually lands

Liquid Clustering doesn't add per-row write overhead the way an extra B-tree index does -- clustering
is applied lazily during writes and background `OPTIMIZE` maintenance, not synchronously on every
insert. The cost instead shows up as **wasted background compute**: every clustered table
consumes optimization cycles reorganizing files toward its declared keys, whether or not that
reorganization measurably helps any real query. Cluster a table nobody filters selectively and
you're paying for file reorganization that produces exactly the query-performance improvement you'd
expect from optimizing nothing -- zero, because there's no selective query pattern for it to serve.

## Three tables that don't need it

- **Small tables.** A dimension table with a few thousand rows already fits in one or two files;
  data skipping has nothing meaningful to skip. Clustering a table this size is spending background
  compute to optimize a scan that was already fast.
- **Tables scanned in full on every query.** If every query against a table reads the entire thing
  -- a small lookup table joined into every fact-table query, for instance -- there's no selective
  filter for clustering to make cheaper. Clustering only pays off when queries can skip a meaningful
  fraction of files, and a full-scan access pattern skips nothing by definition.
- **Low-cardinality, non-selective columns chosen out of habit.** Clustering by a boolean-like
  `is_active` flag or a column with three distinct values doesn't give data skipping enough
  granularity to matter -- the technique needs enough distinct values that "skip the files that
  don't match" actually excludes a useful fraction of the table.

## The right test before adding a clustering key

Ask the same question the previous lecture's workload-inventory approach implies, applied as a
gate rather than a suggestion: **does a real, recurring query in this table's actual workload
filter selectively on this column?** If the honest answer is "maybe, for a report someone might
build eventually," that's not yet a reason to cluster -- `CLUSTER BY AUTO` exists precisely for
that uncertainty, and it costs nothing extra to enable versus guessing wrong on a manually pinned
key.

{: .important }
> Liquid Clustering removing the old "every index has write overhead" tradeoff doesn't mean physical
> design decisions became free -- it means the cost moved to background optimization compute instead
> of write-path latency. Treat "does this table's real query pattern justify clustering at all" as
> the first question, before "which columns," and most small or fully-scanned tables in a migrated
> estate should stay unclustered rather than clustered by default.

The next lecture turns this judgment call into a repeatable checklist -- the physical design
decision card -- so the choice doesn't have to be re-litigated from scratch for every table in the
estate.

<!-- prevnext:start -->

---

| [&larr; Previous: Choosing Cluster Keys from Query Patterns]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/choosing-cluster-keys-from-query-patterns/' | relative_url }}) | [Next: The Physical Design Decision Card &rarr;]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/the-physical-design-decision-card/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

