---
title: "Stored-Procedure Dependency Graphs and Table Heat Maps"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/stored-procedure-dependency-graphs-and-table-heat-maps/
read_minutes: 4
---

# Stored-Procedure Dependency Graphs and Table Heat Maps
{: .no_toc }

*Estimated read: 4 min*

Bottleneck data tells you *what's* expensive. It doesn't tell you *what breaks* if you touch it.
For that you need two structural artifacts most legacy estates have never had drawn out explicitly:
a stored-procedure **dependency graph** and a table-level **heat map**.

**The dependency graph.** Every mature Oracle or SQL Server system accumulates procedures that call
other procedures, which call others still, often three or four layers deep, with the occasional
circular reference where procedure A calls B which, under some condition, calls back into A. None of
this is visible from reading any single procedure in isolation -- you only see it by querying the
system catalog for object dependencies and rendering the result as a graph:

```sql
-- Oracle: direct procedure-to-procedure and procedure-to-table dependencies
SELECT
  name        AS calling_object,
  referenced_name AS called_object,
  referenced_type
FROM all_dependencies
WHERE type = 'PROCEDURE'
  AND owner = 'LEGACY_SCHEMA'
ORDER BY name;
```

Render the output with any graph tool -- Graphviz is enough for a first pass -- and two things jump
out immediately that a flat procedure list never shows: **hub procedures** that dozens of others
call (migrate these first and carefully, since every dependent breaks if the translation is wrong),
and **orphaned procedures** with no incoming calls from anything else in the schema, which are
migration candidates for outright retirement rather than translation. Finding five or six procedures
nobody calls anymore is a common and genuinely useful outcome of this exercise -- it's scope you get
to remove from the project, not add to it.

**The table heat map.** Where the dependency graph shows structural coupling, the heat map shows
actual runtime load: cross-reference the object-access data from AWR's Segment Reads, DBQL's
`DBQLObjTbl`, or Query Store's plan-level object references (previous lecture) against every table in
scope, and you get a matrix of *access frequency* × *data volume* × *number of dependent procedures*.
Plot it and four quadrants fall out naturally:

| | Low dependent procedures | High dependent procedures |
|---|---|---|
| **Low access frequency** | Low priority -- migrate opportunistically, or retire | Migrate carefully but not urgently; logic-heavy, traffic-light |
| **High access frequency** | Migrate early; likely a good re-platform candidate | Highest priority *and* highest risk -- the core of your critical path |

That bottom-right quadrant -- hot tables with heavy procedural dependency -- is where the bulk of
migration risk concentrates in almost every engagement, and it's exactly the set of objects the
next lecture's workload inventory needs to flag as requiring the most scrutiny before anyone commits
to a cutover date.

{: .important }
> Build the dependency graph and heat map *before* you start translating anything with Lakebridge.
> A transpiler will happily convert a hub procedure in isolation, but only the graph tells you that
> forty other procedures depend on its exact output signature -- information you need before you
> change so much as a return type.

With structure and load both mapped, the last piece is turning this into something a steering
committee can actually act on: the workload inventory, covered next.

<!-- prevnext:start -->

---

| [&larr; Previous: Teradata DBQL and SQL Server Query Store: Same Job, Different Dialect]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/teradata-dbql-and-sql-server-query-store-same-job-different-dialect/' | relative_url }}) | [Next: The Workload Inventory: Your Source of Truth &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/the-workload-inventory-your-source-of-truth/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

