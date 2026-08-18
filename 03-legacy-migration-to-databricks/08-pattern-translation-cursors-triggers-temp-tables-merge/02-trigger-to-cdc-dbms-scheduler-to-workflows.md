---
title: "Trigger to CDC, DBMS_SCHEDULER to Workflows"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/trigger-to-cdc-dbms-scheduler-to-workflows/
read_minutes: 3
---

# Trigger to CDC, DBMS_SCHEDULER to Workflows
{: .no_toc }

*Estimated read: 3 min*

Two Oracle constructs, a row-level `AFTER UPDATE` trigger and a `DBMS_SCHEDULER` job, solve
completely different problems -- one reacts to a write, the other runs on a clock -- but both share a
failure mode on a legacy platform: their execution is invisible until something breaks. Neither has
a queryable run history, a retry policy you can inspect, or a dependency graph you can view without
reading source code. Both replacements on Databricks fix that same problem from opposite directions.

## Trigger to Change Data Feed

An Oracle row-level trigger fires *inside* the transaction that wrote the row, executing arbitrary
PL/SQL as a side effect of the write itself. **Delta Lake has no equivalent mechanism, deliberately**
-- a Delta table doesn't execute code on write. Instead, **[Change Data
Feed](https://docs.databricks.com/aws/en/delta/delta-change-data-feed)** (CDF) makes every row-level
change (insert, update, delete) queryable as its own stream of events, and downstream logic consumes
that stream explicitly instead of being triggered implicitly.

```sql
-- Enable once per table
ALTER TABLE orders SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

-- Batch: read what changed between two versions
SELECT * FROM table_changes('orders', 4, 7)
WHERE _change_type IN ('insert', 'update_postimage');
```

```python
# Streaming: react continuously, the way a trigger would have
(spark.readStream.format("delta")
      .option("readChangeFeed", "true")
      .table("orders")
      .filter("_change_type != 'delete'")
      .writeStream.table("orders_audit"))
```

The shift is from *implicit, hidden* execution (a trigger nobody sees fire) to an *explicit,
inspectable* stream (a change feed anyone can query with a `SELECT`). The cascading-trigger graph
from the Autopsy section maps directly onto a chain of CDF-driven Lakeflow Declarative Pipeline
flows, each one reading the previous table's change feed instead of being invoked by an
implicit `AFTER UPDATE`.

## DBMS_SCHEDULER to Lakeflow Jobs

`DBMS_SCHEDULER` runs a PL/SQL block on a cron-like schedule, with retry and dependency behavior
configured through scheduler attributes that live in the database catalog, disconnected from the
job's actual logic. **Lakeflow Jobs** replace it with a task graph where scheduling, retries, and
dependencies are declared alongside the tasks themselves:

```yaml
resources:
  jobs:
    nightly_order_processing:
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
      tasks:
        - task_key: apply_orders
          notebook_task:
            notebook_path: /Repos/migration/apply_daily_orders
          max_retries: 3
        - task_key: refresh_reporting
          depends_on:
            - task_key: apply_orders
          notebook_task:
            notebook_path: /Repos/migration/refresh_reporting_tables
```

The dependency (`refresh_reporting` won't start until `apply_orders` succeeds) and the retry policy
are both visible in the job definition itself -- and every run produces a queryable history: start
time, duration, retry count, and failure reason, all inspectable in the Jobs UI or via the API,
where a `DBMS_SCHEDULER` job's failure history typically lives buried in a scheduler log table few
people ever query.

{: .important }
> The real win in both replacements isn't functional parity -- it's visibility. A trigger and a
> scheduled job that both "work" in the legacy system are still a debugging liability if nobody can
> see what they did without reading source code after something breaks. CDF and Lakeflow Jobs make
> that history a first-class, queryable part of the platform instead of something you reconstruct
> from logs after an incident.

Temp tables are next -- the third legacy construct with no first-class Databricks equivalent, and
two different replacements depending on how long the intermediate result needs to live.

<!-- prevnext:start -->

---

| [&larr; Previous: Cursor to Set-Based DataFrame Ops]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/cursor-to-set-based-dataframe-ops/' | relative_url }}) | [Next: Temp Table to CTE or Lakeflow Declarative Pipeline &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/temp-table-to-cte-or-lakeflow-declarative-pipeline/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

