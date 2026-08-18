---
title: "Teradata DBQL and SQL Server Query Store: Same Job, Different Dialect"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/teradata-dbql-and-sql-server-query-store-same-job-different-dialect/
read_minutes: 3
---

# Teradata DBQL and SQL Server Query Store: Same Job, Different Dialect
{: .no_toc }

*Estimated read: 3 min*

Not every legacy estate is Oracle. If yours is Teradata or SQL Server, the previous lecture's
technique still applies -- you're still hunting for top-offender SQL, wait/resource bottlenecks, and
hot objects -- but the instrumentation you read to get there has a different name and a different
shape.

**Teradata: DBQL (Database Query Log).** Teradata's equivalent of AWR isn't a single report; it's a
family of system tables (`DBQLogTbl`, `DBQLSQLTbl`, `DBQLStepTbl`, `DBQLObjTbl`) that log every query
that ran, its full SQL text, its execution steps, and the objects it touched -- assuming query
logging was turned on for the users or accounts you care about, which is worth confirming before you
assume you have historical coverage. Where AWR gives you a point-in-time snapshot diff, DBQL gives
you a queryable log you can aggregate over any window you choose:

```sql
SELECT
  UserName,
  QueryBand,
  COUNT(*) AS query_count,
  SUM(AMPCPUTime) AS total_cpu_time,
  SUM(TotalIOCount) AS total_io
FROM DBQLogTbl
WHERE StartTime >= CURRENT_DATE - 30
GROUP BY UserName, QueryBand
ORDER BY total_cpu_time DESC;
```

`AMPCPUTime` (CPU consumed across Teradata's AMPs -- the parallel processing units behind each unit
of work) and `TotalIOCount` are your ranking signals, playing the same role AWR's Top SQL and Segment
Reads play for Oracle. `DBQLObjTbl` is what you join against to answer "which tables does this
workload actually touch" -- the Teradata-side input to the heat map you'll build in the next lecture.

**SQL Server: Query Store.** Query Store is closer in spirit to AWR than DBQL is -- it's a built-in
feature (enabled per-database, off by default in most on-prem estates, so check before you assume
history exists) that continuously captures query text, execution plans, and runtime statistics,
browsable through SSMS or queried directly against its system views:

```sql
SELECT TOP 20
    qt.query_sql_text,
    SUM(rs.count_executions) AS total_executions,
    SUM(rs.avg_duration * rs.count_executions) AS total_duration_us,
    SUM(rs.avg_logical_io_reads * rs.count_executions) AS total_logical_reads
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
GROUP BY qt.query_sql_text
ORDER BY total_duration_us DESC;
```

Query Store additionally tracks **plan regressions** -- the same query suddenly getting a worse
execution plan after a statistics update or index change -- which has no direct Oracle or Teradata
analogue in this lecture's scope, but is worth knowing about if a "bottleneck" in your data turns out
to be a plan that regressed six months ago rather than a workload that was always heavy.

| | Oracle AWR | Teradata DBQL | SQL Server Query Store |
|---|---|---|---|
| Form | Point-in-time report | Queryable log tables | Queryable system views |
| Ranking signal | Elapsed time, CPU, wait events | AMPCPUTime, TotalIOCount | Duration, logical reads |
| Object-level detail | Segments by reads | `DBQLObjTbl` | Plan-level object references |
| Needs enabling? | On by default (licensed feature) | Query logging must be enabled per user/account | Off by default per-database in most on-prem installs |

{: .important }
> Whatever the source platform, the discipline is identical: rank workloads by actual resource
> consumption over a representative window, not by row count or table size. The tool name changes;
> the question -- "what is this system actually spending its cycles on" -- doesn't.

With bottleneck data in hand from whichever of these three you're working against, the next lecture
turns raw query logs into something more structural: a dependency graph of what calls what, and a
heat map of which tables carry the load.

<!-- prevnext:start -->

---

| [&larr; Previous: Reading Oracle AWR Reports to Find the Real Bottlenecks]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/reading-oracle-awr-reports-to-find-the-real-bottlenecks/' | relative_url }}) | [Next: Stored-Procedure Dependency Graphs and Table Heat Maps &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/stored-procedure-dependency-graphs-and-table-heat-maps/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

