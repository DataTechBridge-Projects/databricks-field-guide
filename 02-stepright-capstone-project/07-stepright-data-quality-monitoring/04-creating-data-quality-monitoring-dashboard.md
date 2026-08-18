---
title: "Creating Data Quality Monitoring Dashboard"
parent: "StepRight - Data Quality Monitoring"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/07-stepright-data-quality-monitoring/creating-data-quality-monitoring-dashboard/
read_minutes: 8
---

# Creating Data Quality Monitoring Dashboard
{: .no_toc }

*Estimated read: 8 min*

Four views exist, queryable and verified. This lecture turns them into an actual **Databricks
Dashboard** -- SQL datasets built directly on the views, and line, bar, and table visualizations
the audience Lecture 1 designed for can actually read at a glance.

## From view to dataset

A Databricks Dashboard's building block is a **dataset**: a named SQL query the dashboard runs on
its own refresh schedule, decoupled from any specific visualization. Every dataset here is close to
a one-line pass-through of a Lecture 3 view, with only the date-range framing a chart actually needs:

```sql
-- Dataset: quarantine_rate_30d
SELECT source, event_date, quarantine_rate
FROM dev.step_right.dq_quarantine_rate
WHERE event_date >= current_date() - INTERVAL 30 DAYS
ORDER BY event_date;

-- Dataset: orphans_by_check_7d
SELECT check_name, source, sum(orphan_count) AS total_orphans
FROM dev.step_right.dq_referential_orphans
WHERE event_date >= current_date() - INTERVAL 7 DAYS
GROUP BY check_name, source;

-- Dataset: expectations_latest_run
SELECT table_name, expectation_name, passed_records, failed_records
FROM dev.step_right.dq_expectations_summary
WHERE event_date = current_date();
```

Scoping each dataset's date range at the SQL level, not in the visualization config, keeps the
underlying query readable and testable on its own -- a `WHERE event_date >= ...` clause is exactly
the kind of thing worth eyeballing directly before trusting a chart built on top of it.

## Three visualizations, three different questions

| Visualization | Dataset | Chart type | Question it answers |
|---|---|---|---|
| Quarantine rate trend | `quarantine_rate_30d` | Line, one series per source | Is any source's rate drifting upward over the last 30 days |
| Orphans by check | `orphans_by_check_7d` | Bar, grouped by `check_name` | Which specific referential check is generating the most rejects this week |
| Today's expectations | `expectations_latest_run` | Table | Did every structural expectation pass on the most recent run |

A line chart is the right shape for `quarantine_rate_30d` specifically because the *trend* is the
point -- Lecture 1's whole motivation was catching a rate creeping toward 5% before it crosses the
line, which a single day's number, however presented, can't show. A bar chart suits
`orphans_by_check_7d` because it's a comparison across discrete categories, not a time series. The
expectations table stays a table, deliberately -- pass/fail across several named checks reads more
clearly as rows than as any chart type would force it into.

## Building it: dataset first, then visualization

1. **Databricks workspace -> New -> Dashboard.**
2. **Add a dataset** for each of the three queries above, one at a time -- each becomes reusable
   across multiple visualizations if a later need arises, rather than being locked to one chart.
3. **Add a visualization**, select the dataset, choose the chart type from the table above, and map
   columns (`event_date` to the X-axis, `quarantine_rate` to Y, `source` as the series grouping,
   for the line chart).
4. **Set the dashboard refresh schedule** -- daily, aligned with Section 5's job schedule, since a
   more frequent refresh would just re-run the same queries against data that hasn't changed since
   the last scheduled pipeline run.

## What the finished dashboard actually shows

Picture three widgets on one page. The line chart: seven colored lines, one per bronze source,
mostly hugging the 1-2% band Section 1's Faker generator injects by design, with a flat dashed
reference line at 5% running across the top -- healthy data reads as "everything well below the
dashed line, every day." The bar chart: three or four bars, `unknown_customer_id` typically the
tallest given it's the referential check with the highest deliberate injection rate, the other
checks noticeably shorter. The table: seven or eight rows, one per structural expectation, every
`failed_records` column reading zero on a clean day -- a single non-zero cell is immediately
visible against a column of zeros in a way it wouldn't be buried in a paragraph of prose.

## Dashboard as code

A Databricks Dashboard can be exported as a `.lvdash.json` definition -- the dashboard's datasets,
visualizations, and layout, serialized to a single file that can live in the Git folder alongside
everything else this project has built since Section 1:

```bash
databricks lakeview export --dashboard-id <dashboard-id> > resources/dq_dashboard.lvdash.json
```

Committing this file means the dashboard's definition is versioned and reviewable the same way
`pipeline.yml` and `orchestration_job.yml` already are -- a chart's SQL changing is a diff someone
can review, not a silent click-through edit in the UI that leaves no trail. Section 8's Asset
Bundle covers deploying this file as a bundle resource, the same `resources/` folder Section 5's
job definition and Section 2's pipeline definitions already live in.

## Sharing and permissions

The audience Lecture 1 designed for -- governance, operations, merchandising -- needs to see this
dashboard without needing write access to `dev.step_right`, or even SQL warehouse compute of their
own. Databricks Dashboards support sharing to specific users or groups with **view-only** access,
running on the dashboard owner's credentials or a designated service principal rather than each
viewer's own permissions -- the practical mechanism that lets a non-technical stakeholder open a
link and see current data without ever being granted `SELECT` on the underlying tables directly.
This is the same credential-vending instinct Unity Catalog applies elsewhere in this project,
applied here to a dashboard instead of a Delta Sharing recipient.

## Adding a threshold reference line

The quarantine rate line chart is far more useful with `dq_check`'s own 5% threshold drawn
alongside it -- most Databricks visualization types support a **reference line** at a fixed value,
turning "is this number bad" from a mental calculation into something visually obvious the moment
a series crosses it. Set it once, at 0.05, matching `QUARANTINE_THRESHOLD` from Section 5's
`dq_check.py` exactly -- a dashboard threshold line that doesn't match the job's actual gate value
would show "healthy" visually while the job itself is genuinely failing, or the reverse.
{: .important }

## Why this dashboard, and not a generic BI tool pointed at the same tables

A third-party BI tool could technically query these same four views -- but keeping the dashboard
inside Databricks means one less credential to provision, one less network hop out of Unity
Catalog's governance boundary, and no separate extract-and-load step just to get quarantine counts
in front of a browser. For a monitoring surface whose entire value is freshness -- catching a
drifting quarantine rate quickly -- adding a scheduled export into a separate BI platform would
undercut the exact responsiveness this section exists to provide.

## Common mistakes

- **Building a chart directly against a raw bronze table "just this once."** Every exception to the
  views-only rule from Lecture 1 is one more place a future schema change can silently break a
  chart nobody remembers depends on it -- there's no dashboard need this section's four views don't
  already cover.
- **Setting the dashboard's refresh schedule more frequently than the underlying pipeline runs.**
  Refreshing every hour against data that only changes once a day burns query compute for
  identical results every single time between actual pipeline runs.

## What's next

Lecture 5 closes the loop: an alert that watches `dq_quarantine_rate` continuously and notifies a
human the moment a source crosses the threshold this chart only shows after the fact.

<!-- prevnext:start -->

---

| [&larr; Previous: Prepare DQ Queries for Expectations, Quarantine and Orphans]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/prepare-dq-queries-for-expectations-quarantine-and-orphans/' | relative_url }}) | [Next: Setup a Real Time Data Quality Monitoring and Alert &rarr;]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/setup-a-real-time-data-quality-monitoring-and-alert/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

