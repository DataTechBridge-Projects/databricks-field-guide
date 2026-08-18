---
title: "Prepare DQ Queries for Expectations, Quarantine and Orphans"
parent: "StepRight - Data Quality Monitoring"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/07-stepright-data-quality-monitoring/prepare-dq-queries-for-expectations-quarantine-and-orphans/
read_minutes: 8
---

# Prepare DQ Queries for Expectations, Quarantine and Orphans
{: .no_toc }

*Estimated read: 8 min*

Three signals, three base views: `dq_expectations_summary` (event log), `dq_quarantine_summary`
(bronze quarantine tables), and `dq_referential_orphans` (foreign-key checks) -- plus one derived
view, `dq_quarantine_rate`, built on top of the second. Every visualization Lecture 4 builds
queries one of these four -- nothing else.

## `dq_expectations_summary`: from the event log, across every monitored table

```sql
CREATE OR REPLACE VIEW dev.step_right.dq_expectations_summary AS
SELECT
    date_trunc('day', timestamp) AS event_date,
    'silver_orders' AS table_name,
    exp.name AS expectation_name,
    exp.passed_records,
    exp.failed_records
FROM event_log(TABLE(dev.step_right.silver_orders))
LATERAL VIEW explode(details:flow_progress.data_quality.expectations) AS exp
WHERE event_type = 'flow_progress'

UNION ALL

SELECT
    date_trunc('day', timestamp) AS event_date,
    'bronze_orders_tagged' AS table_name,
    exp.name AS expectation_name,
    exp.passed_records,
    exp.failed_records
FROM event_log(TABLE(dev.step_right.bronze_orders_tagged))
LATERAL VIEW explode(details:flow_progress.data_quality.expectations) AS exp
WHERE event_type = 'flow_progress';
```

One `UNION ALL` branch per expectation-bearing table, each hardcoding its own `table_name` literal
-- the same explicit, one-branch-per-source shape [Section 5, Lecture 2]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-pipeline-data-quality-and-health-check-implementation/' | relative_url }})'s
per-source quarantine loop used, adapted here to SQL since a view can't loop the way a Python
script can.

## `dq_quarantine_summary`: one row per source, per day, across all seven

```sql
CREATE OR REPLACE VIEW dev.step_right.dq_quarantine_summary AS
SELECT 'customers' AS source, date(_ingested_at) AS event_date, count(*) AS quarantine_count
FROM dev.step_right.bronze_customers_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'orders', date(_ingested_at), count(*)
FROM dev.step_right.bronze_orders_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'order_items', date(_ingested_at), count(*)
FROM dev.step_right.bronze_order_items_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'products', date(_ingested_at), count(*)
FROM dev.step_right.bronze_products_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'inventory', date(_ingested_at), count(*)
FROM dev.step_right.bronze_inventory_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'clickstream', date(_ingested_at), count(*)
FROM dev.step_right.bronze_clickstream_quarantine GROUP BY date(_ingested_at)
UNION ALL
SELECT 'fulfillment', date(_ingested_at), count(*)
FROM dev.step_right.bronze_fulfillment_quarantine GROUP BY date(_ingested_at);
```

This is the persisted, queryable version of exactly what `dq_check` computes ad hoc on every job
run -- the difference is that a view holds no state itself and can be queried across *every*
historical `run_date` at once, which is what makes a trend line (Lecture 4) possible in the first
place. `dq_check` still owns the pass/fail decision for a single run; this view owns the history
`dq_check` was never designed to retain.

## `dq_referential_orphans`: foreign keys that don't resolve

```sql
CREATE OR REPLACE VIEW dev.step_right.dq_referential_orphans AS
SELECT
    'orders' AS source,
    'unknown_customer_id' AS check_name,
    date(_ingested_at) AS event_date,
    count(*) AS orphan_count
FROM dev.step_right.bronze_orders_quarantine
WHERE array_contains(_dq_failed_rules, 'unknown_customer_id')
GROUP BY date(_ingested_at)

UNION ALL

SELECT
    'order_items',
    'unknown_order_id',
    date(_ingested_at),
    count(*)
FROM dev.step_right.bronze_order_items_quarantine
WHERE array_contains(_dq_failed_rules, 'unknown_order_id')
GROUP BY date(_ingested_at)

UNION ALL

SELECT
    'order_items',
    'unknown_product_id',
    date(_ingested_at),
    count(*)
FROM dev.step_right.bronze_order_items_quarantine
WHERE array_contains(_dq_failed_rules, 'unknown_product_id')
GROUP BY date(_ingested_at);
```

`array_contains(_dq_failed_rules, ...)` is what separates a genuine referential-integrity failure
from every other reason a row might be quarantined -- `bronze_orders_quarantine` also holds rows
that failed `missing_order_id`, which has nothing to do with a foreign key not resolving. Filtering
specifically to the referential check names, rather than counting the whole quarantine table, is
what makes this view answer "are foreign keys resolving" and not just "how much got quarantined
overall" -- that broader question already belongs to `dq_quarantine_summary`.

## A fourth, derived view: quarantine rate, not just raw counts

A raw quarantine count alone is hard to read without context -- 41 quarantined orders means
something very different on a 200-order day than on a 2,000-order day. `dq_quarantine_summary`
holds counts only; a small derived view turns those into the rate a trend chart actually needs:

```sql
CREATE OR REPLACE VIEW dev.step_right.dq_quarantine_rate AS
SELECT
    q.source,
    q.event_date,
    q.quarantine_count,
    v.valid_count,
    q.quarantine_count / (q.quarantine_count + v.valid_count) AS quarantine_rate
FROM dev.step_right.dq_quarantine_summary q
JOIN (
    SELECT 'orders' AS source, date(_ingested_at) AS event_date, count(*) AS valid_count
    FROM dev.step_right.bronze_orders_valid GROUP BY date(_ingested_at)
    UNION ALL
    SELECT 'order_items', date(_ingested_at), count(*)
    FROM dev.step_right.bronze_order_items_valid GROUP BY date(_ingested_at)
    UNION ALL
    SELECT 'clickstream', date(_ingested_at), count(*)
    FROM dev.step_right.bronze_clickstream_valid GROUP BY date(_ingested_at)
) v ON q.source = v.source AND q.event_date = v.event_date;
```

This is the same `quarantine_rate = quarantine / (valid + quarantine)` formula `dq_logic.py`'s
`quarantine_rate()` function computes in Python (Section 6, Lecture 4) -- expressed here in SQL
because a dashboard dataset needs to be a query, not a Python function call. Keeping both versions
consistent matters: if this formula and the unit-tested Python version ever drift apart, the
dashboard and the job's own gate could disagree about whether a given day was actually healthy.

## Why separate views instead of one combined view

A single view attempting to hold expectations, quarantine counts, and referential orphans together
would need three structurally different `GROUP BY` shapes and three different source tables forced
into one schema -- exactly the kind of one-clever-combined-query Section 5, Lecture 3 already
argued against for `report.py`'s bronze/silver/gold counts. Four separate, narrowly-scoped views is
worth the small duplication of `date(_ingested_at)` grouping logic across them.

## Why `array_contains` needs the exact rule name, not a substring match

`_dq_failed_rules` holds exact rule-name strings, the same ones `tag_quality` writes in Section 2 --
`array_contains(_dq_failed_rules, 'unknown_customer_id')`, not a `LIKE '%unknown%'` pattern match.
A substring match would also catch a hypothetical future rule like `unknown_shipping_region`, silently
folding an unrelated check into what's supposed to be a narrowly-scoped referential-integrity view.
Exact matching against the rule-name vocabulary Section 2 already established keeps
`dq_referential_orphans` honest about exactly which checks it's counting.

## Verifying the views

```sql
SELECT * FROM dev.step_right.dq_quarantine_summary ORDER BY event_date DESC, source;
SELECT * FROM dev.step_right.dq_referential_orphans ORDER BY event_date DESC;
SELECT * FROM dev.step_right.dq_expectations_summary ORDER BY event_date DESC;
SELECT * FROM dev.step_right.dq_quarantine_rate WHERE quarantine_rate > 0.05 ORDER BY event_date DESC;
```

The last query is worth running specifically -- it's `dq_check`'s own 5% threshold, expressed as a
`WHERE` filter over history instead of a single run's pass/fail check, and should return no rows
under normal conditions for the same reason `dq_check` passes on an ordinary day.

Each should return roughly what Section 2, Lecture 6's known injection rates predict -- `orders`
and `order_items` quarantine counts landing near 1-2% of daily volume, the referential orphan view
showing exactly the `unknown_customer_id`, `unknown_order_id`, and `unknown_product_id` breakdown
that make up most of that quarantine total.

## Views, not materialized views, and why that's the right call here

Every `dq_` view here is a plain SQL `VIEW`, computed fresh on every query, not a materialized view
refreshed on a pipeline schedule. That's a deliberate difference from the gold layer's
materialized views (Section 4): a dashboard visualization querying `dq_quarantine_rate` wants
whatever the underlying quarantine and valid tables currently hold at query time, not a snapshot
that's only as fresh as the last `steprightproject-silver-gold` run. These views cost more per
query than a materialized view would, but at StepRight's data volume that cost is negligible, and
the freshness a plain view guarantees matters more for a monitoring dashboard than it does for a
gold table finance queries once a day.

## Naming views with a `dq_` prefix, deliberately

Every view in this lecture starts with `dq_`, distinguishing it at a glance from `silver_` and
`gold_` tables in the same `dev.step_right` schema -- a small convention, but one that matters the
moment someone unfamiliar with the project runs `SHOW TABLES` and needs to tell a reporting table
meant for business consumers apart from a monitoring view meant for the data quality dashboard
alone. Consistent prefixing is a cheap substitute for documentation that scales better than a
README no one reads before running a query.

## What's next

Four views, ready to query. Lecture 4 turns them into an actual dashboard -- SQL datasets built
directly on top of these views, and line, bar, and table visualizations reading from them.

<!-- prevnext:start -->

---

| [&larr; Previous: Setup Pipeline Event Logs for Expectations Monitoring]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/setup-pipeline-event-logs-for-expectations-monitoring/' | relative_url }}) | [Next: Creating Data Quality Monitoring Dashboard &rarr;]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/creating-data-quality-monitoring-dashboard/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

