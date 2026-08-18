---
title: "Job Data Quality and Health Reporting in the Job Logs"
parent: "StepRight - Orchestration and Job Scheduling"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-data-quality-and-health-reporting-in-the-job-logs/
read_minutes: 10
---

# Job Data Quality and Health Reporting in the Job Logs
{: .no_toc }

*Estimated read: 10 min*

`dq_check` answers one narrow question -- did bronze pass the gate. Nobody checking on this job at
9am wants to run seven separate `SELECT COUNT(*)` queries across three layers just to confirm
yesterday's run actually worked end to end. This lecture builds `report`, the task that runs last
and prints exactly that picture -- bronze, silver, gold, quarantines, one run date -- directly into
the job run's own logs, no dashboard, no separate query tool required.

## Why the job's own logs, and not a notebook someone has to remember to open

A Databricks Jobs run page already shows a full stdout/stderr capture for every task, one click
from the job's run history -- the same place a failure's stack trace shows up. Printing a plain-text
summary there means the person checking on last night's run finds the whole picture -- did it
succeed, and *what did it actually produce* -- in the exact place they were already looking, rather
than in a separate notebook, dashboard, or query they'd need to know to go run. This is the
Databricks-native version of the end-of-job summary a legacy Autosys or Control-M chain used to
mail out or drop into a shared log directory -- except here it costs nothing extra to wire up, since
the Jobs UI is already capturing every print statement.

## What the report covers, layer by layer

| Layer | What gets counted | Scope |
|---|---|---|
| Bronze | `_valid` and `_quarantine` row counts, all 7 sources | `date(_ingested_at) = run_date` |
| Silver | Row counts, all 4 tables | Current version only (`__END_AT IS NULL`) for the 3 SCD Type 2 tables; full count for `silver_products` |
| Gold | Row counts, all 5 materialized views | Full count -- gold tables are rollups, not per-run appends, so "how many rows exist" is the meaningful number, not "how many rows changed today" |

Silver's current-version filter is the same rule Section 4, Lecture 1 built `gold_daily_revenue`
around -- `silver_orders`, `silver_order_items`, and `silver_customers` all carry full SCD Type 2
history, so a naive `COUNT(*)` on any of them counts every historical version of every row, not the
number of *distinct* customers, orders, or line items currently on file. `silver_products`, built
as a plain deduplicated table with no `AUTO CDC` history, needs no such filter -- the same
exception Section 4 relied on throughout the gold layer.

## Building the report

```python
# report.py
import argparse
from databricks.sdk.runtime import spark

BRONZE_SOURCES = ["customers", "orders", "order_items", "products", "inventory", "clickstream", "fulfillment"]
SILVER_SCD2_TABLES = ["silver_orders", "silver_customers", "silver_order_items"]
GOLD_TABLES = [
    "gold_daily_revenue",
    "gold_customer_360",
    "gold_product_performance",
    "gold_clickstream_funnel",
    "gold_fulfillment_health",
]

def count(table, where=None):
    query = f"SELECT count(*) FROM dev.step_right.{table}"
    if where:
        query += f" WHERE {where}"
    return spark.sql(query).collect()[0][0]

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--run_date", required=True)
    args = parser.parse_args()

    print("=" * 60)
    print(f"StepRight Pipeline Run Summary -- {args.run_date}")
    print("=" * 60)

    print("\n-- BRONZE --")
    total_valid, total_quarantine = 0, 0
    for source in BRONZE_SOURCES:
        valid = count(f"bronze_{source}_valid", f"date(_ingested_at) = '{args.run_date}'")
        quarantine = count(f"bronze_{source}_quarantine", f"date(_ingested_at) = '{args.run_date}'")
        total_valid += valid
        total_quarantine += quarantine
        print(f"  {source:<14} valid={valid:>6}  quarantine={quarantine:>4}")
    print(f"  {'TOTAL':<14} valid={total_valid:>6}  quarantine={total_quarantine:>4}")

    print("\n-- SILVER --")
    for table in SILVER_SCD2_TABLES:
        current = count(table, "__END_AT IS NULL")
        print(f"  {table:<20} current_rows={current:>6}")
    products = count("silver_products")
    print(f"  {'silver_products':<20} current_rows={products:>6}")

    print("\n-- GOLD --")
    for table in GOLD_TABLES:
        rows = count(table)
        print(f"  {table:<28} rows={rows:>6}")

    print("\n" + "=" * 60)

if __name__ == "__main__":
    main()
```

```yaml
tasks:
  - task_key: report
    depends_on: [{task_key: run_transformation}]
    python_wheel_task:
      package_name: "steprightproject"
      entry_point: "report"
      parameters: ["--run_date", "{{job.parameters.run_date}}"]
```

Same `count()` helper, called once per table with a different `where` clause -- a small function
covering three structurally different counting rules (date-scoped bronze, current-version-only
SCD2 silver, full-count gold) rather than three near-duplicate blocks of SQL, the same
one-helper-many-callers shape Section 2's `read_cdc_bronze` and Section 4's per-consumer gold
builders both leaned on.

## What it prints

```text
============================================================
StepRight Pipeline Run Summary -- 2026-08-17
============================================================

-- BRONZE --
  customers      valid=  1204  quarantine=  12
  orders         valid=  2033  quarantine=  41
  order_items    valid=  2033  quarantine=  38
  products       valid=    86  quarantine=   0
  inventory      valid=    86  quarantine=   0
  clickstream    valid=   511  quarantine=   3
  fulfillment    valid=   312  quarantine=   0
  TOTAL          valid=  6265  quarantine=  94

-- SILVER --
  silver_orders        current_rows= 48210
  silver_customers      current_rows=  6842
  silver_order_items    current_rows= 71344
  silver_products       current_rows=    86

-- GOLD --
  gold_daily_revenue           rows=  9184
  gold_customer_360            rows=  6842
  gold_product_performance     rows=    86
  gold_clickstream_funnel      rows=   511
  gold_fulfillment_health      rows= 48210

============================================================
```

One glance answers the questions someone checking this run actually has: did today's batch land
(`BRONZE` total matches roughly what a normal day looks like), did quarantine stay in the expected
1-2% range `dq_check` already gated on, and did every downstream layer grow in a way consistent
with what came in. `gold_customer_360` matching `silver_customers`' row count exactly, for
instance, is a quick sanity signal that Section 4's one-row-per-customer design is holding, without
needing to go run the verification queries Section 4's lectures each ended with by hand.

## This is a standalone task, not a consumer of `dq_check`'s task values

`report` re-queries bronze and quarantine counts itself rather than reading them from `dq_check`
via `dbutils.jobs.taskValues.get` -- [Part 1, Section 10's task-value lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/passing-parameters-to-python-script-task-task-value-repair-runs/' | relative_url }})
introduced that mechanism for exactly this kind of cross-task data sharing, and it would work here
too. The simpler design wins for a report task specifically: `report` also needs silver and gold
counts `dq_check` never computed, so it's going to query the warehouse directly regardless: adding
a task-value dependency on `dq_check` for the one overlapping metric (bronze counts) would just be
one more thing that breaks if `dq_check`'s output shape ever changes, for no real benefit.

## What `report` deliberately doesn't do

`report` prints; it doesn't page anyone, chart a trend, or track a threshold over time -- on
purpose. A print-to-logs summary answers "what happened on this specific run" for someone already
looking at it, which is a different job from proactive alerting on a metric drifting slowly across
many runs, or a dashboard tracking `quarantine_rate` week over week to spot a slow degradation
`dq_check`'s single-run 5% threshold would never catch on its own. That's real scope for a later
section, not this task: [Section 7]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/' | relative_url }})
builds exactly that trend-tracking layer on top of the same counts `report` already knows how to
compute, the same rolling-average pattern [Part 1, Section 10's multi-task DAG lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/multi-task-dags-control-flow-retries-and-backfill/' | relative_url }})
forward-referenced. Keeping `report` narrow here -- one run, one summary, into the logs -- is also what makes it
trivial to run standalone for a single historical date, covered next.

## Running `report` standalone against a historical date

Because `report` takes `run_date` as a plain argument rather than reading `dbutils.jobs.taskValues`
from a specific upstream run, it doesn't need the job at all to answer "what did last Tuesday's
pipeline actually produce" -- it can run directly from a cluster or a local terminal against the
workspace:

```bash
python report.py --run_date 2026-08-10
```

This matters for anyone investigating an incident days after the fact: reprinting a historical
run's summary doesn't require re-running the job, re-triggering `dq_check`, or touching
`run_transformation` at all -- `report` only ever reads, never writes, so running it against an
arbitrary past date is always safe.

## Why `count()` runs one query per table instead of one big join

It would be possible to write a single sprawling query joining bronze, silver, and gold counts
into one result row -- but that couples three layers with genuinely different scopes (`_ingested_at`
for bronze, `__END_AT IS NULL` for three of four silver tables, no filter at all for gold) into one
query someone has to untangle every time a table gets added or removed. Twelve small, independently
readable `count()` calls cost nothing extra at this data volume and stay obviously correct one
table at a time -- the same reasoning Section 2's `read_cdc_bronze` helper and Lecture 2's per-source
loop both leaned on: many simple, uniform calls over one clever combined one.

## Common mistakes

- **Wrapping every count in a `try`/`except` that swallows errors and prints "N/A."** A report task
  that silently prints "N/A" for a table that's actually missing or inaccessible hides a real
  problem behind a summary that looks complete at a glance -- if a count query fails, that failure
  belongs in the task's own error output, not papered over.
- **Reporting gold row counts without ever comparing them to silver's.** The numbers only mean
  something in relation to each other -- `gold_customer_360` should track `silver_customers`
  closely; a gold table's row count drifting far from what feeds it is usually the first visible
  sign something upstream broke, and a report that prints raw numbers without that context makes
  the drift easy to miss.
{: .important }

## What's next

Three of the job's four tasks are built. Lecture 4 wires `run_ingestion`, `dq_check`,
`run_transformation`, and `report` into the actual job resource, adds the schedule, and runs the
whole thing end to end for the first time.

<!-- prevnext:start -->

---

| [&larr; Previous: Job Pipeline Data Quality and Health Check Implementation]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-pipeline-data-quality-and-health-check-implementation/' | relative_url }}) | [Next: Creating the Orchestration Job &rarr;]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

