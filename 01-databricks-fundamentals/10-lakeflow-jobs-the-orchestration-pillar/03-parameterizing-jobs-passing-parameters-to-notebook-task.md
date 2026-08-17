---
title: "Parameterizing jobs - Passing Parameters to Notebook Task"
parent: "Lakeflow Jobs - The Orchestration Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/parameterizing-jobs-passing-parameters-to-notebook-task/
read_minutes: 16
---

# Parameterizing jobs - Passing Parameters to Notebook Task
{: .no_toc }

*Estimated read: 16 min*

Section 4 introduced notebook **widgets** as interactive parameters. This lecture closes the loop:
how those same widgets receive values from a scheduled job, plus dynamic value references that
make a job's parameters context-aware without hardcoding.

## Job parameters vs. task parameters

- **Job parameters** -- defined once, at the job level, available to every task in the job.
- **Task parameters** -- specific to one task, can reference job parameters or be entirely
  independent.

```yaml
resources:
  jobs:
    daily_orders_job:
      parameters:
        - name: run_date
          default: "{{job.trigger.time.iso_date}}"
      tasks:
        - task_key: process_orders
          notebook_task:
            notebook_path: /Repos/steprightproject/notebooks/process_orders
            base_parameters:
              run_date: "{{job.parameters.run_date}}"
              environment: "prod"
```

## Receiving parameters as widgets

Inside the notebook, this is exactly Section 4's `dbutils.widgets` API -- no separate "job
parameter" API to learn:

```python
dbutils.widgets.text("run_date", "", "Run Date")
dbutils.widgets.text("environment", "dev", "Environment")

run_date = dbutils.widgets.get("run_date")
environment = dbutils.widgets.get("environment")
```

**Key term:** this is the payoff of Section 4's earlier note that widgets serve both interactive
use and job parameterization identically -- when this notebook runs as a scheduled job task, the
values come from the job's parameter configuration instead of a human clicking a dropdown, with
zero code changes required in the notebook itself.
{: .important }

## Dynamic value references

Rather than hardcoding a run date, **dynamic value references** compute values from the job's own
runtime context:

```text
{{job.trigger.time.iso_date}}        -- the date this run was triggered
{{job.id}}                            -- this job's ID
{{job.run_id}}                        -- this specific run's ID
{{job.parameters.<name>}}             -- another job parameter's value
```

```yaml
base_parameters:
  run_date: "{{job.trigger.time.iso_date}}"
  run_id: "{{job.run_id}}"
```

This means a job scheduled to run daily automatically passes *that day's* date into the notebook,
without you updating a hardcoded parameter value before every run -- directly comparable to a
legacy scheduler's built-in `${execution_date}`-style macro, if your prior ETL tool had one.

## A realistic parameterized notebook task

```python
# process_orders notebook
dbutils.widgets.text("run_date", "", "Run Date")
dbutils.widgets.text("environment", "dev", "Environment")

run_date = dbutils.widgets.get("run_date")
environment = dbutils.widgets.get("environment")

catalog = "prod" if environment == "prod" else "dev"

orders_df = (spark.table(f"{catalog}.silver.orders")
    .filter(f"order_date = '{run_date}'"))

print(f"Processing {orders_df.count()} orders for {run_date} in {catalog}")
```

One notebook, driven entirely by parameters -- runs identically whether triggered manually with
test values, or by the scheduled job with `{{job.trigger.time.iso_date}}` substituted
automatically.

## Testing parameterized notebooks before scheduling them

Run the notebook interactively first, manually setting widget values via the UI, to confirm the
logic behaves correctly for a few different `run_date`/`environment` combinations -- before
trusting a schedule to supply values you haven't actually exercised. This is the same "test before
you trust the automation" discipline from the previous lecture's job-verification step, applied
one level down at the individual task's parameter handling.

The next lecture covers the same parameterization problem for **Python script tasks** (which don't
have `dbutils.widgets` available) and introduces **task values** and **repair runs**.

<!-- prevnext:start -->

---

| [&larr; Previous: Your first job - pipeline task, schedule, and job-level config]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/your-first-job-pipeline-task-schedule-and-job-level-config/' | relative_url }}) | [Next: Passing Parameters to Python Script Task, Task Value, Repair Runs &rarr;]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/passing-parameters-to-python-script-task-task-value-repair-runs/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
