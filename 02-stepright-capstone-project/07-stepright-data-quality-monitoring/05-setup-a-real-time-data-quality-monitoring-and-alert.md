---
title: "Setup a Real Time Data Quality Monitoring and Alert"
parent: "StepRight - Data Quality Monitoring"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/07-stepright-data-quality-monitoring/setup-a-real-time-data-quality-monitoring-and-alert/
read_minutes: 7
---

# Setup a Real Time Data Quality Monitoring and Alert
{: .no_toc }

*Estimated read: 7 min*

A dashboard someone has to remember to open only helps if they actually open it. This closing
lecture adds a **Databricks SQL Alert** that watches `dq_quarantine_rate` on its own and notifies a
human the moment a source crosses threshold -- without anyone needing to check the dashboard at all.

## "Real time" here means something specific -- worth being honest about it

This isn't streaming, and it doesn't need to be. StepRight's data changes once a day, when Section
5's job runs `run_ingestion` and `run_transformation` on their 3 AM schedule -- there's nothing to
alert on more frequently than that, since no new quarantine data exists between runs. "Real time"
in this lecture means the alert checks *as soon as new data exists to check*, immediately after
each day's run finishes, not once a week when someone happens to remember to look at the
dashboard. That's a meaningfully different, and more honest, claim than implying continuous,
sub-second monitoring of a batch pipeline that fundamentally doesn't produce sub-second data.
{: .important }

## The alert query

```sql
-- Alert: quarantine_rate_breach
SELECT source, event_date, quarantine_rate
FROM dev.step_right.dq_quarantine_rate
WHERE event_date = current_date()
  AND quarantine_rate > 0.05;
```

A Databricks SQL Alert evaluates a query on a schedule and fires when the result set is non-empty
(or against a specific column threshold, depending on the condition type configured) -- this query
returns rows only on a day where at least one source's rate actually breached 5%, which is exactly
the condition worth paging someone about.

## Scheduling the alert to run after the job, not on its own clock

```text
Alert schedule: Daily at 3:45 AM America/New_York
```

Fifteen minutes after Section 5's job kicks off at 3 AM is enough buffer for `run_ingestion`,
`dq_check`, and `run_transformation` to have finished on a normal day -- scheduling the alert
query to run *before* the pipeline has actually produced that day's data would just evaluate
against yesterday's numbers, missing the point entirely. This is the same source-timing discipline
[Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
applied to picking the job's own 3 AM schedule, applied one level up here.

## Notification destinations

```text
Alert notifications:
  - Email: data-eng-team@company.com
  - Slack webhook: #stepright-data-quality
```

Databricks SQL Alerts support email and webhook-based destinations (Slack, PagerDuty, a generic
webhook) configured once per alert -- the same `data-eng-team@company.com` address Section 5's job
already notifies on failure is worth reusing here too, so the team watching for pipeline failures
and the team watching for data quality drift are, deliberately, the same people looking at the
same inbox.

## A second alert: watching the trend, not just today's breach

`quarantine_rate_breach` catches a single bad day. A slower-moving problem -- a source's rate
creeping from 1% toward 4% over three weeks, never once crossing 5% -- needs a different query,
comparing today against a recent baseline rather than against a fixed threshold:

```sql
-- Alert: quarantine_rate_trending_up
WITH baseline AS (
    SELECT source, avg(quarantine_rate) AS avg_rate_last_14d
    FROM dev.step_right.dq_quarantine_rate
    WHERE event_date BETWEEN current_date() - 15 AND current_date() - 1
    GROUP BY source
),
today AS (
    SELECT source, quarantine_rate AS today_rate
    FROM dev.step_right.dq_quarantine_rate
    WHERE event_date = current_date()
)
SELECT t.source, t.today_rate, b.avg_rate_last_14d
FROM today t
JOIN baseline b ON t.source = b.source
WHERE t.today_rate > b.avg_rate_last_14d * 2;
```

This is the same rolling-average comparison [Part 1, Section 10's multi-task DAG lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/multi-task-dags-control-flow-retries-and-backfill/' | relative_url }})
forward-referenced this exact section for -- "today's rate more than double the last two weeks'
average" catches a source quietly degrading well before it ever reaches the fixed 5% line
`quarantine_rate_breach` and `dq_check` both watch. The two alerts genuinely complement each other:
one catches sudden, obvious breaches; the other catches slow drift a fixed threshold is
structurally unable to see.

## How this differs from `dq_check`'s gate, one more time

| | `dq_check` (Section 5) | This alert |
|---|---|---|
| Runs | As a job task, blocking `run_transformation` | Independently, after the job finishes |
| Scope | That run's data only | That day's data, queryable against full history |
| Effect on the pipeline | Skips downstream tasks on failure | None -- notifies only, never blocks anything |
| Audience | Engineers watching the Jobs UI | Whoever's on the alert's notification list |

`dq_check` protects the pipeline from processing bad data forward; this alert protects a human from
finding out about bad data too late. Both check the same 5% number, deliberately, but they exist
for genuinely different reasons and neither one makes the other redundant.

## Testing an alert that's designed to almost never fire

An alert that's correctly configured but has never actually fired is untested, the same problem
[Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
solved by deliberately pushing `bronze_clickstream_valid`'s quarantine rate over threshold on
purpose. The same test data trick works here: seed one day's batch with an inflated injection
rate, let the job run, and confirm both `quarantine_rate_breach` actually fires and the
notification actually lands in the configured Slack channel -- not just that the alert's SQL
returns the expected rows when run manually, which proves the query is right but says nothing
about whether the notification wiring behind it works at all.

## Common mistakes

- **Setting the alert threshold looser than `dq_check`'s, "to reduce noise."** If the alert only
  fires at a higher rate than the job's own gate already failed at, every alert is stale news by
  the time it arrives -- `dq_check` will have already failed the job hours earlier. The alert
  threshold should match `dq_check`'s exactly, not sit above it.
- **Alerting on every dashboard refresh instead of once per day.** An alert query scheduled hourly
  against data that only changes once daily just re-fires the same notification repeatedly for one
  underlying event, training recipients to ignore it -- the same "notify on the exception, not the
  routine" principle Section 5, Lecture 4 applied to job-level email notifications.

## Section wrap-up

Five lectures, four views, one dashboard, one alert -- StepRight's data quality story is now
visible over time, not just pass/fail on a single run. Section 8 closes out the capstone: real
integration tests against a deployed environment, and the Asset Bundle packaging that ships every
pipeline, job, and dashboard this project has built as one deployable unit.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating Data Quality Monitoring Dashboard]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/creating-data-quality-monitoring-dashboard/' | relative_url }}) | [Next: StepRight - Integration Testing, Packaging and Deployment &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

