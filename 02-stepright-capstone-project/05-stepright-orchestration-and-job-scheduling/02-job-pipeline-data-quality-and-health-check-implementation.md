---
title: "Job Pipeline Data Quality and Health Check Implementation"
parent: "StepRight - Orchestration and Job Scheduling"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-pipeline-data-quality-and-health-check-implementation/
read_minutes: 6
---

# Job Pipeline Data Quality and Health Check Implementation
{: .no_toc }

*Estimated read: 6 min*

Lecture 1 decided *where* the quality gate sits -- between `run_ingestion` and
`run_transformation`. This lecture builds it: a Python task that reads what bronze's quarantine
pattern (Section 2, Lecture 6) actually found for today's run, and decides whether transformation
gets to run at all.

## What `dq_check` reads

Section 2 left seven pairs of bronze tables -- `bronze_customers_valid` /
`bronze_customers_quarantine`, and six more sources following the same `_valid` / `_quarantine`
split, each row tagged with `_ingested_at`. `dq_check` scopes every count to `run_date`, the same
job parameter Lecture 1 introduced, rather than an all-time cumulative total that would dilute a
bad day's quarantine spike against months of clean history:

```sql
SELECT
    'orders' AS source,
    (SELECT count(*) FROM dev.step_right.bronze_orders_valid WHERE date(_ingested_at) = :run_date) AS valid_count,
    (SELECT count(*) FROM dev.step_right.bronze_orders_quarantine WHERE date(_ingested_at) = :run_date) AS quarantine_count
```

Run once per source (a loop over the same seven-name list Section 2's quarantine pattern already
uses), this gives `dq_check` a valid/quarantine count pair for every bronze table, scoped to
exactly the rows `run_ingestion` produced this run.

## The threshold decision, per source

```python
# dq_check.py
import argparse
import sys
from databricks.sdk.runtime import spark

SOURCES = ["customers", "orders", "order_items", "products", "inventory", "clickstream", "fulfillment"]
QUARANTINE_THRESHOLD = 0.05  # 5%

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--run_date", required=True)
    args = parser.parse_args()

    failures = []
    for source in SOURCES:
        valid = spark.sql(f"""
            SELECT count(*) FROM dev.step_right.bronze_{source}_valid
            WHERE date(_ingested_at) = '{args.run_date}'
        """).collect()[0][0]
        quarantined = spark.sql(f"""
            SELECT count(*) FROM dev.step_right.bronze_{source}_quarantine
            WHERE date(_ingested_at) = '{args.run_date}'
        """).collect()[0][0]

        total = valid + quarantined
        rate = quarantined / total if total > 0 else 0.0
        print(f"{source}: {quarantined}/{total} quarantined ({rate:.1%})")

        if rate > QUARANTINE_THRESHOLD:
            failures.append(f"{source}: {rate:.1%} exceeds {QUARANTINE_THRESHOLD:.0%} threshold")

    if failures:
        print("DQ CHECK FAILED:")
        for f in failures:
            print(f"  - {f}")
        sys.exit(1)

    print("DQ CHECK PASSED")

if __name__ == "__main__":
    main()
```

```yaml
tasks:
  - task_key: dq_check
    depends_on: [{task_key: run_ingestion}]
    python_wheel_task:
      package_name: "steprightproject"
      entry_point: "dq_check"
      parameters: ["--run_date", "{{job.parameters.run_date}}"]
```

Same `python_wheel_task` shape [Part 1, Section 10, Lecture 4]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/passing-parameters-to-python-script-task-task-value-repair-runs/' | relative_url }})
used for `reconcile.py` -- `run_date` arrives as a command-line argument, not a notebook widget,
because a wheel task has no notebook execution context to hold one.

## Why `sys.exit(1)` is the whole mechanism

A `python_wheel_task` fails when its entry point exits non-zero or raises an unhandled exception --
that's it, no separate "mark this task failed" API call needed. `sys.exit(1)` here is what turns a
quarantine spike into an actual failed task in the Jobs UI, which is what makes `run_transformation`,
declared with `run_if: ALL_SUCCESS` against `dq_check`, skip instead of run.

## Why per-source, not one blended rate

Averaging all seven sources into a single number would let a badly broken `bronze_clickstream`
(say, 40% quarantined because an upstream schema change slipped through) hide inside six healthy
sources averaging the blended rate back down under 5%. Checking each source independently means
one bad source fails the gate on its own, regardless of how clean the other six look -- the same
reasoning Section 2's quarantine pattern applied per-source rather than pooling every bronze table
into one combined valid/quarantine split.

## Picking 5%, not an arbitrary number

Section 1, Lecture 5's Faker generator injects roughly 1% `unknown_customer_id` rows and 2%
orphaned `order_items` by design -- Section 2, Lecture 6 already confirmed quarantine counts land
close to those known injection rates against batch zero. A 5% threshold sits comfortably above
that expected baseline noise while still catching a genuine regression (a schema change, a broken
upstream feed) an order of magnitude worse than what the synthetic data intentionally seeds. A
threshold set *at* the expected baseline would false-positive on ordinary day-to-day variance; one
set at 50% would never catch anything short of a near-total outage.

## What a passing and a failing run each look like in the job logs

A clean day prints seven quiet lines and exits zero:

```text
customers: 12/1204 quarantined (1.0%)
orders: 41/2033 quarantined (2.0%)
order_items: 38/2033 quarantined (1.9%)
products: 0/86 quarantined (0.0%)
inventory: 0/86 quarantined (0.0%)
clickstream: 3/511 quarantined (0.6%)
fulfillment: 0/312 quarantined (0.0%)
DQ CHECK PASSED
```

A day where `bronze_clickstream`'s upstream event schema silently changed looks different --
notice only the affected source crosses the line, exactly the outcome the per-source check was
built to produce:

```text
customers: 12/1204 quarantined (1.0%)
orders: 41/2033 quarantined (2.0%)
order_items: 38/2033 quarantined (1.9%)
products: 0/86 quarantined (0.0%)
inventory: 0/86 quarantined (0.0%)
clickstream: 219/511 quarantined (42.9%)
fulfillment: 0/312 quarantined (0.0%)
DQ CHECK FAILED:
  - clickstream: 42.9% exceeds 5% threshold
```

`run_transformation` never starts on this run -- silver and gold stay exactly as clean as
yesterday's successful run left them, rather than absorbing a day of malformed clickstream data
into a materialized view finance and growth both eventually query.

## The legacy-world equivalent

A hand-built Talend or Informatica chain solved the same problem with a **staging-then-promote**
pattern: land data into a staging table, run a row-count or checksum validation step against a
control table, and only truncate-and-load the real target table if validation passed -- a manual,
often SQL-procedure-based gate bolted onto the front of the next job step. `dq_check` is that same
checkpoint instinct, expressed as a first-class Lakeflow Job task instead of a stored procedure
someone has to remember to call, with its pass/fail outcome visible in the Jobs UI's task graph
rather than buried in a control table only the job's author knows to query.

## Common mistakes

- **Filtering by `current_date()` instead of the `run_date` parameter.** A job re-triggered the
  next morning for a missed run, or manually re-run for a specific historical date, needs to check
  *that* date's quarantine counts, not whatever date the check happens to execute on.
- **Catching the exception and logging a warning instead of calling `sys.exit(1)`.** A caught
  exception leaves the task showing `SUCCESS` in the Jobs UI, which means `run_if: ALL_SUCCESS`
  on `run_transformation` still fires -- the entire point of the gate silently defeated by
  well-intentioned error handling.
{: .important }

## What's next

`dq_check` decides pass or fail; it doesn't summarize *what happened* across the whole run.
Lecture 3 builds `report`, the task that runs after `run_transformation` succeeds and prints the
full bronze/silver/gold picture into the job's own logs.

<!-- prevnext:start -->

---

| [&larr; Previous: Design the Orchestration and Job Flow]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/design-the-orchestration-and-job-flow/' | relative_url }}) | [Next: Job Data Quality and Health Reporting in the Job Logs &rarr;]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-data-quality-and-health-reporting-in-the-job-logs/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

