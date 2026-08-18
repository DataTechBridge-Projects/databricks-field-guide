---
title: "Check Your Knowledge"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you've learned from this section before moving on to the 3-R decision and TCO.

1. In the iceberg assessment framing, what does the "visible 20%" typically represent?
   A. The procedures and orchestration logic
   B. Table counts and storage volume, easily pulled from catalog metadata
   C. Undocumented behavioral quirks
   D. Access patterns over the last 30 days

2. Which of the following is explicitly named as part of the submerged "80%" that a simple table/TB count misses?
   A. Scheduler dependency chains and orchestration logic
   B. The Databricks workspace URL
   C. The cluster's Photon version
   D. The Unity Catalog metastore ID

3. In an Oracle AWR report, which section is described as the best signal for migration-priority ranking?
   A. Alert log error counts
   B. Top SQL by Elapsed Time / by CPU
   C. Listener configuration
   D. Tablespace fragmentation report

4. What is the Databricks-side system table described as playing a similar role to AWR, once workloads have landed on the lakehouse?
   A. `system.billing.usage`
   B. `system.query.history`
   C. `system.access.audit`
   D. `system.compute.clusters`

5. In Teradata, which system table is specifically called out as the source for which tables a workload actually touches?
   A. `DBQLogTbl`
   B. `DBQLSQLTbl`
   C. `DBQLObjTbl`
   D. `DBQLStepTbl`

6. What does SQL Server Query Store track that has no direct equivalent called out for Oracle AWR or Teradata DBQL in this section?
   A. Row counts per table
   B. Plan regressions -- the same query getting a worse execution plan after a change
   C. Storage cost per schema
   D. Unity Catalog grants

7. In the stored-procedure dependency graph, what does an "orphaned procedure" (no incoming calls) typically indicate?
   A. It is the most critical procedure in the system and must be migrated first
   B. It is a candidate for outright retirement rather than translation
   C. It cannot be queried through `all_dependencies`
   D. It is always a trigger, never a stored procedure

8. In the table heat map's four quadrants, which combination represents the highest migration priority and highest risk?
   A. Low access frequency, low dependent procedures
   B. High access frequency, low dependent procedures
   C. Low access frequency, high dependent procedures
   D. High access frequency, high dependent procedures

9. What are the three verdicts a workload inventory assigns to each object?
   A. Approve, Reject, Escalate
   B. Lift, Redesign, Retire
   C. Rehost, Re-platform, Re-architect
   D. Bronze, Silver, Gold

10. Why does the workload inventory require a `source_query_id` column tracing each cell back to a rerunnable query?
    A. It is a Unity Catalog requirement for external tables
    B. Without it, the inventory decays into an unverifiable spreadsheet that can't be defended or reproduced later
    C. It is required by Lakebridge's assessment API
    D. It determines the read_minutes value shown on each page

## Answer Key

1. **B** -- The visible 20% is the easily-counted table/storage metadata; logic, orchestration, access patterns, and behavior make up the submerged 80%.
2. **A** -- Orchestration (scheduler dependency chains) is named alongside logic, access patterns, and behavior as part of the submerged 80%.
3. **B** -- Top SQL by Elapsed Time / by CPU is the ranked list of what's actually burning compute, the key migration-priority signal from AWR.
4. **B** -- `system.query.history` is the Databricks-side, account-wide query log described as AWR's lakehouse-side analogue.
5. **C** -- `DBQLObjTbl` logs the objects each query touched, the Teradata-side input to the table heat map.
6. **B** -- Query Store's plan-regression tracking (a query getting a worse plan after a stats/index change) is called out as having no named Oracle/Teradata equivalent in this section.
7. **B** -- An orphaned procedure with no incoming calls is a strong candidate for retirement, reducing migration scope rather than adding to it.
8. **D** -- High access frequency combined with high dependent-procedure count concentrates the most migration risk and the highest priority.
9. **B** -- Lift, Redesign, and Retire are the three verdicts, feeding directly into the 3-R decision in the next section.
10. **B** -- Traceable source queries are what let anyone reproduce or challenge a verdict later; without them the inventory becomes an unverifiable opinion.

<!-- prevnext:start -->

---

| [&larr; Previous: The Workload Inventory: Your Source of Truth]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/the-workload-inventory-your-source-of-truth/' | relative_url }}) | [Next: The 3-R Decision and the TCO That Convinces the CFO &rarr;]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

