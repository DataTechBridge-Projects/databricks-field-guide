---
title: "Reading Oracle AWR Reports to Find the Real Bottlenecks"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/reading-oracle-awr-reports-to-find-the-real-bottlenecks/
read_minutes: 3
---

# Reading Oracle AWR Reports to Find the Real Bottlenecks
{: .no_toc }

*Estimated read: 3 min*

If your legacy platform is Oracle, you already have the single best source of truth for what's
actually expensive to run: the **AWR (Automatic Workload Repository)** report. It's the same report
your DBA has pulled for years to diagnose a slow batch window -- and it's exactly what you need now
to figure out which workloads actually deserve migration priority, instead of guessing from table
row counts.

An AWR report is a snapshot-to-snapshot diff of the instance's internal performance statistics. Run
it across a representative window -- ideally covering both an OLTP peak and the overnight batch
window, since they stress completely different parts of the system -- and three sections matter most
for migration scoping:

- **Top SQL by Elapsed Time / by CPU.** This is the ranked list of what's actually burning compute,
  not what you assume is heavy because it "feels big." A three-line `UPDATE` running inside a
  cursor loop ten million times will outrank a single well-written 200-line report query every time
  -- and that ranking is exactly the signal that tells you where re-architecture will pay off most.
- **Wait Event statistics.** Whether the instance is bottlenecked on I/O (`db file sequential read`,
  `db file scattered read`), CPU, or contention (`enq: TX - row lock contention`) tells you whether
  a workload's pain is a hardware/config problem that migration alone fixes, or a design problem
  (row-by-row processing, missing indexes, serial execution) that migration alone won't fix --
  because you'll reproduce the same design on the new platform if you lift-and-shift it as-is.
- **Segments by Logical/Physical Reads.** The tables and indexes actually being hammered, cross-
  referenced against your table heat map (next lecture) to confirm the objects you think are "hot"
  are the ones the instance agrees are hot.

{: .important }
> Read AWR data as a *ranking* tool, not an absolute-number tool. "This procedure accounts for 40%
> of total elapsed time across the batch window" is an actionable migration-priority signal.
> "This procedure took 4.2 seconds" in isolation tells you almost nothing about whether it deserves
> re-architecture effort.

The Databricks-side equivalent you'll eventually rely on for the same purpose, once workloads have
landed on the lakehouse, is the **[`system.query.history` system
table](https://docs.databricks.com/aws/en/admin/system-tables/query-history)** -- an account-wide,
queryable record of every query run against SQL warehouses and serverless compute, with duration,
bytes scanned, and spill statistics per query. Where AWR requires parsing a static report, system
tables let you run the same "top offenders" analysis as a plain SQL query:

```sql
SELECT
  statement_type,
  COUNT(*) AS execution_count,
  SUM(total_duration_ms) AS total_duration_ms,
  AVG(total_duration_ms) AS avg_duration_ms
FROM system.query.history
WHERE start_time >= now() - INTERVAL 7 DAYS
GROUP BY statement_type
ORDER BY total_duration_ms DESC;
```

Getting comfortable reading AWR now pays off twice: it gives you an honest priority order for this
migration, and it primes the same analytical instinct you'll use against `system.query.history` to
keep the new platform's costs in check long after cutover -- the subject of the FinOps sections later
in this part.

<!-- prevnext:start -->

---

| [&larr; Previous: The Iceberg Assessment: 20% Data, 80% Everything Else]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/the-iceberg-assessment-20-data-80-everything-else/' | relative_url }}) | [Next: Teradata DBQL and SQL Server Query Store: Same Job, Different Dialect &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/teradata-dbql-and-sql-server-query-store-same-job-different-dialect/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

