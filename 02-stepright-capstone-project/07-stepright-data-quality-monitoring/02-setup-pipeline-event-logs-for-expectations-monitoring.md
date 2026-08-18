---
title: "Setup Pipeline Event Logs for Expectations Monitoring"
parent: "StepRight - Data Quality Monitoring"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/07-stepright-data-quality-monitoring/setup-pipeline-event-logs-for-expectations-monitoring/
read_minutes: 5
---

# Setup Pipeline Event Logs for Expectations Monitoring
{: .no_toc }

*Estimated read: 5 min*

Lecture 1 decided expectations would layer onto bronze and silver as a second, report-only
monitoring signal. This lecture adds them, and shows how to actually query what they produce from
the pipeline's event log.

## Adding report-only expectations

`@dp.expect` -- bare, not `_or_drop` or `_or_fail` -- tags a row's pass/fail result into the event
log without dropping or blocking anything, [Part 1, Section 9]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-bronze-layer-auto-loader-as-a-streaming-table/' | relative_url }})
already introduced for exactly this "structural, not business-logic" case:

```python
# transformations/bronze_quality.py (addition)
@dp.table(name="bronze_orders_tagged", comment="bronze_orders with structural DQ tags")
@dp.expect("has_ingestion_timestamp", "_ingested_at IS NOT NULL")
@dp.expect("has_source_identifier", "_source_file IS NOT NULL OR _ingested_at IS NOT NULL")
def bronze_orders_tagged():
    ...  # unchanged from Section 2, Lecture 6
```

```python
# transformations/silver_cdc.py (addition)
@dp.table(name="silver_orders", comment="...")
@dp.expect("reasonable_order_total", "order_total IS NULL OR order_total < 100000")
def silver_orders():
    ...  # unchanged from Section 3
```

Every expectation added here checks pipeline plumbing or a sanity bound, never a rule
`tag_business_rules` already owns -- `reasonable_order_total` catches a genuinely malformed value
(a data-entry error six orders of magnitude too large, say), which is a different concern from
`silver_orders`'s existing critical business-rule checks around referential integrity.

## Querying expectation results from the event log

```sql
SELECT
    date_trunc('day', timestamp) AS event_date,
    details:flow_progress.data_quality.expectations AS expectation_results
FROM event_log(TABLE(dev.step_right.silver_orders))
WHERE event_type = 'flow_progress'
ORDER BY timestamp DESC
LIMIT 20;
```

`event_log(TABLE(...))`, the same table-valued function [Part 1, Section 9]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/building-the-bronze-layer-auto-loader-as-a-streaming-table/' | relative_url }})
introduced, returns one row per pipeline event -- filtering to `event_type = 'flow_progress'` scopes
to the events that carry per-flow processing metrics, including `data_quality.expectations`: an
array of `{name, passed_records, failed_records}` per expectation defined on that flow, for that
specific pipeline update.

## Unpacking the nested array

```sql
SELECT
    date_trunc('day', timestamp) AS event_date,
    exp.name AS expectation_name,
    exp.passed_records,
    exp.failed_records
FROM event_log(TABLE(dev.step_right.silver_orders))
LATERAL VIEW explode(details:flow_progress.data_quality.expectations) AS exp
WHERE event_type = 'flow_progress'
ORDER BY event_date DESC;
```

`LATERAL VIEW explode(...)` turns the one-row-per-event, array-valued result into one row per
expectation per event -- the shape a dashboard visualization or a downstream view actually needs,
rather than a nested structure every consumer would have to unpack independently.

## Why this needs no changes to the job from Section 5

The event log updates automatically as a side effect of every pipeline run -- adding `@dp.expect`
constraints changes what gets logged, not how or when the pipeline runs. `run_ingestion` and
`run_transformation` (Section 5) need no modification at all; every run they already trigger now
also populates expectation results, ready for Lecture 3's views to read.

## How many expectations is enough

Two or three per table, scoped to genuine structural sanity checks, not an attempt to replicate
every business rule `tag_business_rules` already covers as an expectation too. Duplicating logic
across both mechanisms means a future rule change has to be made twice, in two different places,
with no guarantee both get updated together -- expectations here stay deliberately narrow:
ingestion metadata presence, an obviously-out-of-range numeric bound, nothing that overlaps with a
check already living in `dq_helpers.py`.

## Common mistakes

- **Reaching for `@dp.expect_or_drop` here out of habit.** Dropping a row over a structural
  expectation would silently remove it from the table entirely, bypassing StepRight's own
  tag-don't-drop quarantine philosophy -- report-only `@dp.expect` is the deliberate choice for
  every constraint this lecture adds, not an oversight.
- **Querying `event_log` per table in a dashboard visualization directly.** One `event_log(TABLE(...))`
  call per bronze and silver table, repeated across every chart, is exactly the raw-query sprawl
  Lecture 1's views-layer decision exists to avoid -- Lecture 3 consolidates this into one
  reusable view.
{: .important }

## What `passed_records` and `failed_records` actually count

Each expectation's counters reset per pipeline update -- `passed_records` and `failed_records` on
a given `flow_progress` event describe only the rows that flow processed *in that specific run*,
not a cumulative total since the table was created. That's exactly what makes the event log usable
for trend analysis in the first place: a `GROUP BY date_trunc('day', timestamp)` over many
historical events (Lecture 3) reconstructs a day-by-day trend precisely because each event already
represents one bounded slice of activity, not a running total that would need to be manually
differenced first.

## Where these constraints actually live in the pipeline definition

`@dp.expect` decorators stack directly above the existing `@dp.table` decorator on the same
function -- Section 2 and Section 3's original transformation files gain a line or two each, not a
new file. `dq_helpers.py`'s pure functions (`tag_quality`, `tag_business_rules`) need no changes at
all; expectations operate at the decorator level, entirely separate from the DataFrame logic those
helpers already handle.

## What's next

Lecture 3 builds the three `dq_` views Lecture 1 designed -- starting with `dq_expectations_summary`,
built directly on top of the query this lecture just wrote.

<!-- prevnext:start -->

---

| [&larr; Previous: Design Your Data Quality Monitoring Requirements]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/design-your-data-quality-monitoring-requirements/' | relative_url }}) | [Next: Prepare DQ Queries for Expectations, Quarantine and Orphans &rarr;]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/prepare-dq-queries-for-expectations-quarantine-and-orphans/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

