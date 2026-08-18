---
title: "Design Your Data Quality Monitoring Requirements"
parent: "StepRight - Data Quality Monitoring"
grand_parent: "StepRight Capstone Project"
nav_order: 1
permalink: /02-stepright-capstone-project/07-stepright-data-quality-monitoring/design-your-data-quality-monitoring-requirements/
read_minutes: 5
---

# Design Your Data Quality Monitoring Requirements
{: .no_toc }

*Estimated read: 5 min*

Section 6 proved the transformation logic is correct against known inputs. Section 5's `dq_check`
gates a single run's quarantine rate against one fixed threshold. Neither answers a question a real
data governance stakeholder actually asks: *is data quality getting worse over time*, and *can
anyone see that without writing SQL*. This lecture designs what closes that gap.

## What `dq_check` deliberately doesn't do

[Part 1, Section 10's multi-task DAG lecture]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/multi-task-dags-control-flow-retries-and-backfill/' | relative_url }})
forward-referenced this exact section for a reason: `dq_check` makes one pass/fail decision per
run, against a hardcoded 5% threshold, visible only in that run's job logs. It can't show whether
`clickstream`'s quarantine rate has crept from 1% to 4% over three weeks -- still under threshold
every single day, so `dq_check` never fires, while the trend itself is exactly the kind of early
warning a governance team wants to see before it crosses 5% and actually breaks something
downstream.

## Three signals, three different sources

| Signal | Source | What it answers |
|---|---|---|
| **Expectations** | SDP's native event log | Is pipeline-level plumbing healthy (nulls in metadata, malformed rows at bronze) |
| **Quarantine counts** | `bronze_*_quarantine` tables (Section 2) | How much data fails StepRight's own business rules, and how that's trending |
| **Referential integrity** | The `unknown_customer_id`-style checks (Sections 2-3) | Are foreign keys resolving -- orphaned rows pointing at records that don't exist |

These aren't three views of the same thing -- they're genuinely different failure modes.
Expectations catch structural plumbing issues (a source stops sending `_source_file` metadata,
say); quarantine counts catch StepRight's own business-rule violations; referential checks catch a
specific, especially damaging failure mode where an order references a customer that plain
row-level validation would miss entirely, since the row itself looks perfectly well-formed.

## Why this needs SDP's native expectations at all, given Section 2 already built quarantine tagging

`@dp.expect` (bare, report-only -- not `_or_drop` or `_or_fail`) logs pass/fail counts directly
into the pipeline's **event log** with zero extra infrastructure -- no new table, no extra write.
Layering a handful of lightweight, structural expectations onto bronze and silver tables gets a
second, independent monitoring signal essentially for free, without touching StepRight's existing
`tag_quality`/`tag_business_rules` quarantine pattern (Sections 2-3) at all. The two mechanisms
stay cleanly separated by purpose: quarantine tagging still owns every *business-rule* decision
this project cares about (that logic is unit-tested, in production, and does real routing);
expectations exist purely to widen what the event log can tell a monitoring dashboard, report-only
and non-blocking by design.
{: .important }

## Design decision: a `dq_` views layer, not raw table queries from the dashboard

Every dashboard visualization Lecture 4 builds queries a small set of purpose-built SQL views --
`dq_expectations_summary`, `dq_quarantine_summary`, `dq_referential_orphans` -- rather than querying
`bronze_*_quarantine` or the event log directly from each chart. A dashboard built straight against
raw tables means every visualization re-derives the same aggregation logic independently, and any
schema change to a bronze table risks silently breaking a chart nobody remembers depends on it. A
views layer is the one place that logic lives, tested and readable on its own, the same
separation-of-concerns instinct Section 6 applied to transformation code, applied here to
reporting queries instead.

## What's out of scope

This section monitors and alerts -- it does not auto-remediate. A quarantine rate crossing an
alert threshold (Lecture 5) notifies a human; it doesn't automatically retry, backfill, or patch
bad data. That boundary is deliberate: automatic remediation of a data quality problem risks
silently masking the exact issue someone needs to actually investigate.

## Who actually looks at this

`dq_check`'s pass/fail lives in the Jobs UI, a place engineers look. This section's dashboard is
built for a different audience -- a data governance stakeholder, an operations lead, or a
merchandising analyst who has no reason to ever open the Jobs UI, but who does want a five-second
answer to "is our data healthy this week" without asking an engineer to run a query for them. That
audience shift is a real design constraint, not just a nice-to-have: it's what pushes Lecture 4
toward visual charts over raw tables, and Lecture 5 toward a notification landing somewhere that
audience actually watches, rather than another job-log entry only engineers would ever see.

## The legacy-world equivalent

A legacy warehouse team solved the same problem with a hand-maintained data quality scorecard --
often a spreadsheet, refreshed manually or by a scheduled export, tracking row counts and reject
rates week over week because no built-in mechanism surfaced that history automatically. Everything
this section builds replaces that spreadsheet with something that updates itself: the event log
populates as a side effect of every pipeline run, the `dq_` views compute directly against live
tables, and the dashboard queries those views on its own refresh schedule -- nobody has to
remember to update anything by hand.

## What's next

Lecture 2 adds the lightweight `@dp.expect` constraints this design calls for and shows how to
query their results from the event log. Lecture 3 builds the three `dq_` views. Lecture 4 turns
them into an actual dashboard. Lecture 5 closes the loop with alerting.

<!-- prevnext:start -->

---

| [&larr; Previous: StepRight - Data Quality Monitoring]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/' | relative_url }}) | [Next: Setup Pipeline Event Logs for Expectations Monitoring &rarr;]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/setup-pipeline-event-logs-for-expectations-monitoring/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

