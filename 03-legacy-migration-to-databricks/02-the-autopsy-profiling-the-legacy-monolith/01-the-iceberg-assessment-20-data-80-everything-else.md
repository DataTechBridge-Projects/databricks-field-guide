---
title: "The Iceberg Assessment: 20% Data, 80% Everything Else"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/the-iceberg-assessment-20-data-80-everything-else/
read_minutes: 3
---

# The Iceberg Assessment: 20% Data, 80% Everything Else
{: .no_toc }

*Estimated read: 3 min*

Ask a legacy DBA to scope a migration and you'll get a number of terabytes, a table count, and a
rough sense of the busiest schemas. That's the visible tip of the **iceberg assessment** -- the part
above the waterline, easy to measure because it's just metadata queries against the catalog. It's
also roughly 20% of what actually determines whether the migration succeeds.

The other 80% is submerged and doesn't show up in a `SELECT COUNT(*) FROM dba_tables` sweep:

- **Logic.** Every stored procedure, trigger, and package that encodes a business rule nobody wrote
  down anywhere else. A 50-table schema with three procedures is a different project than a
  50-table schema with three hundred.
- **Orchestration.** The scheduler jobs, dependency chains, and "run this at 2am after that job
  finishes" logic sitting in Control-M, `DBMS_SCHEDULER`, or a cron table nobody has fully diagrammed.
  You can't migrate the tables without migrating the sequencing that keeps them consistent.
- **Access patterns.** Which of your 4,000 tables are actually queried, by what, and how often. A
  table with a billion rows and zero reads in the last two years is a very different migration
  decision than a hundred-thousand-row table hit by every dashboard refresh.
- **Behavior.** Undocumented quirks -- a report that silently depends on Oracle's default `NULLS
  LAST` sort order, a nightly job that only works because it happens to run before a specific other
  job finishes. These are the things that pass every code review and still break in production the
  week after cutover.

{: .important }
> Budget and timeline estimates built from the visible 20% consistently run over, because the
> submerged 80% is where the actual engineering effort lives. If your assessment phase produced a
> table count and a TB figure and nothing else, you haven't assessed the workload -- you've
> inventoried the storage.

The rest of this section is the toolkit for surfacing the submerged 80% before it surfaces itself
mid-project: reading your platform's own query instrumentation to find real bottlenecks (not assumed
ones), building dependency graphs and table heat maps to see logic and access patterns instead of
guessing at them, and rolling all of it into a workload inventory that becomes the one document every
later decision in this part refers back to. None of this requires exotic tooling -- Oracle AWR,
Teradata DBQL, and SQL Server Query Store are already running on your legacy platform today,
recording exactly the data you need. The next three lectures show you how to read them.

<!-- prevnext:start -->

---

| [&larr; Previous: The Autopsy: Profiling the Legacy Monolith]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/' | relative_url }}) | [Next: Reading Oracle AWR Reports to Find the Real Bottlenecks &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/reading-oracle-awr-reports-to-find-the-real-bottlenecks/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

