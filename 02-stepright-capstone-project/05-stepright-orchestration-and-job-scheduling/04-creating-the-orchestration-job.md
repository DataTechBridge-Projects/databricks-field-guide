---
title: "Creating the Orchestration Job"
parent: "StepRight - Orchestration and Job Scheduling"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/
read_minutes: 8
---

# Creating the Orchestration Job
{: .no_toc }

*Estimated read: 8 min*

Three tasks exist as code. This lecture assembles all four -- `run_ingestion`, `dq_check`,
`run_transformation`, `report` -- into one job resource, adds the schedule, and runs the complete
DAG end to end for the first time.

## The job resource

Following the project structure Section 1 planned (`resources/` holds job and pipeline
definitions), this lives at `resources/orchestration_job.yml`:

```yaml
resources:
  jobs:
    stepright_daily_pipeline:
      name: "StepRight Daily Pipeline"
      schedule:
        quartz_cron_expression: "0 0 3 * * ?"
        timezone_id: "America/New_York"
      email_notifications:
        on_failure: ["data-eng-team@company.com"]
      parameters:
        - name: run_date
          default: "{{job.trigger.time.iso_date}}"
      tasks:
        - task_key: run_ingestion
          pipeline_task:
            pipeline_id: ${var.bronze_pipeline_id}

        - task_key: dq_check
          depends_on:
            - task_key: run_ingestion
          python_wheel_task:
            package_name: "steprightproject"
            entry_point: "dq_check"
            parameters: ["--run_date", "{{job.parameters.run_date}}"]

        - task_key: run_transformation
          depends_on:
            - task_key: dq_check
          run_if: ALL_SUCCESS
          pipeline_task:
            pipeline_id: ${var.silver_gold_pipeline_id}

        - task_key: report
          depends_on:
            - task_key: run_transformation
          python_wheel_task:
            package_name: "steprightproject"
            entry_point: "report"
            parameters: ["--run_date", "{{job.parameters.run_date}}"]
```

Four tasks, three `depends_on` edges, one `run_if` gate -- the exact DAG Lecture 1 designed,
expressed as the same asset-bundle-style job resource [Part 1, Section 10, Lecture 2]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/your-first-job-pipeline-task-schedule-and-job-level-config/' | relative_url }})
introduced. `${var.bronze_pipeline_id}` and `${var.silver_gold_pipeline_id}` reference the two
pipelines Lecture 1 split `steprightproject-bronze-cdc` into -- Section 8's packaging lecture
covers where bundle variables like these actually get defined and substituted per environment
(`dev`, `uat`, `prod`).

## Why 3 AM, and why that's a real decision

`quartz_cron_expression: "0 0 3 * * ?"` isn't an arbitrary time slot -- it needs to run late enough
that whatever upstream system feeds StepRight's CDC changefeed and landing volume has finished its
own overnight batch, but early enough that finance, marketing, and operations have fresh gold
tables before their own morning. A schedule picked without checking source-system timing is a
common way to end up ingesting a partial night's data -- worth confirming against the actual
upstream schedule, not just picking a round number.

## Job-level settings vs. what each task controls

| Setting | Scope here | Why |
|---|---|---|
| `schedule` | Job-level only | One trigger fires the whole DAG -- a schedule per task would leave `dq_check` and `run_transformation` racing each other with no guaranteed order outside `depends_on` |
| `parameters` (`run_date`) | Job-level, read by every task | The one value every task needs to agree on, set once at the top rather than four times |
| `email_notifications` | Job-level default, `on_failure` | A failure anywhere in the DAG -- `run_ingestion`, `dq_check`, or `run_transformation` -- should page the same team; no task here needs a narrower audience |
| `pipeline_id` / `python_wheel_task` | Per-task | Each task's actual unit of work is inherently task-specific -- there's nothing to share at the job level here |

[Part 1, Section 10, Lecture 2]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/your-first-job-pipeline-task-schedule-and-job-level-config/' | relative_url }})
covered this split in general (schedule and notifications job-wide, compute and timeout
overridable per task); this job happens to need no per-task overrides at all, which is itself a
sign the four-task design stayed appropriately simple.

## Verifying `run_date` actually threads through every task

A scheduled run's default (`{{job.trigger.time.iso_date}}`) is easy to trust blindly -- the real
test is a manual run for a specific historical date, the same mechanic
[Part 1, Section 10, Lecture 5's backfill lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/multi-task-dags-control-flow-retries-and-backfill/' | relative_url }})
covered:

```bash
databricks jobs run-now --job-id <job-id> --json '{"job_parameters": {"run_date": "2026-08-10"}}'
```

Confirm `dq_check`'s printed quarantine counts and `report`'s summary both show `2026-08-10`, not
today's date -- if either task's output shows the wrong date, that task is reading
`current_date()` somewhere instead of the `--run_date` argument it was actually passed, a mistake
that only surfaces on a manual or backfilled run, never on an ordinary scheduled one where both
values happen to match.

## Deploying and running

```bash
databricks bundle validate
databricks bundle deploy --target dev
databricks jobs run-now --job-id <job-id>
```

Or through the workspace UI: **Workflows -> Jobs -> stepright_daily_pipeline -> Run now**. Either
path, watch the job's DAG view -- it should show `run_ingestion` complete first, `dq_check` run
and pass, `run_transformation` start only after that, and `report` run last, printing the summary
from Lecture 3 into its own task's log output.

## Testing the gate on purpose

A happy-path run alone doesn't actually prove the gate works -- it proves the job runs when
nothing goes wrong, which was never the risky case. Deliberately push `bronze_clickstream_valid`'s
quarantine rate over 5% for a test run (Section 1, Lecture 5's Faker generator can seed a batch
with a higher injection rate for exactly this purpose), then trigger the job again and confirm two
things: `dq_check` fails with a clear message naming `clickstream`, and `run_transformation` shows
as **Skipped**, not **Failed**, in the DAG view -- `run_if: ALL_SUCCESS` skipping a downstream task
looks different in the UI from that task actually running and erroring, and knowing which one
happened matters when someone's debugging a 3 AM failure alert at 8 AM.

## Notifications beyond failure

`email_notifications.on_failure` covers the case that actually needs a human -- but Lakeflow Jobs
also supports `on_success` and `on_start`, worth resisting here rather than adding by default.
A daily success email for a job running once a night trains its recipients to stop reading it
within a week; a failure notification, by contrast, is rare enough by design (`dq_check`'s gate
should mean most failures never reach `run_transformation` at all) that it stays meaningful every
time it fires. Notify on the exception, not on the routine.

## Common mistakes

- **Forgetting `run_if: ALL_SUCCESS` on `run_transformation`.** Without it, `depends_on` alone only
  guarantees ordering -- `run_transformation` would still execute after `dq_check` regardless of
  whether `dq_check` passed or failed, silently defeating the entire gate Lecture 1 through 3 were
  built around.
- **Pointing both pipeline tasks at the same `pipeline_id`.** This is the one mistake that would
  make Lecture 1's whole pipeline-splitting decision meaningless -- `run_ingestion` and
  `run_transformation` need to reference the two distinct pipelines from the split, not the
  original combined one.
{: .important }

## Why Repair run is safe here

A partial failure -- say, `run_transformation` fails after `dq_check` already passed -- doesn't
need the whole DAG rerun from `run_ingestion`. [Part 1, Section 10's Repair run lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/passing-parameters-to-python-script-task-task-value-repair-runs/' | relative_url }})
covered the mechanic in general and its one real caveat: a repair reruns each unsuccessful task
from the beginning, so it's only safe when that task's own logic is idempotent. Both pipeline
tasks here qualify -- Section 3's `AUTO CDC` merges and Section 4's materialized view refreshes are
both re-runnable against the same `run_date` without duplicating anything, which is exactly what
lets a StepRight on-call engineer repair a failed `run_transformation` at 8 AM without worrying
about double-counted revenue in `gold_daily_revenue`.

## What Section 8 changes here, and what it doesn't

This job resource is deploy-ready today, but `${var.bronze_pipeline_id}` and
`${var.silver_gold_pipeline_id}` are still placeholders -- Section 8 is where `databricks.yml`
actually defines those variables per target (`dev`, `uat`, `prod`) and where `resources/` stops
being deployed by hand and starts being deployed through a CI/CD pipeline. Nothing about the DAG
itself, the gate, or the schedule changes between now and then; only *how* this file gets from a
laptop into a real environment does.

## Section wrap-up

Sections 2 through 4 built the tables; this section built the job that keeps them current without
anyone clicking "run." A scheduled trigger, a real quality gate between ingestion and
transformation, and a plain-text summary landing in the job's own logs -- StepRight now runs the
way a production pipeline actually should. Section 6 turns to a different kind of confidence:
verifying the transformation logic itself is correct, with unit tests that don't need a live
cluster or real data to run.

<!-- prevnext:start -->

---

| [&larr; Previous: Job Data Quality and Health Reporting in the Job Logs]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-data-quality-and-health-reporting-in-the-job-logs/' | relative_url }}) | [Next: StepRight - Unit Testing &rarr;]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

