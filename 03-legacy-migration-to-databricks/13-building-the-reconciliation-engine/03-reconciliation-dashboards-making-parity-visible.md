---
title: "Reconciliation Dashboards: Making Parity Visible"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/reconciliation-dashboards-making-parity-visible/
read_minutes: 3
---

# Reconciliation Dashboards: Making Parity Visible
{: .no_toc }

*Estimated read: 3 min*

A Delta audit table with millions of row-level diff records is a great source of truth and a terrible thing to hand a steering committee. The reconciliation engine's second job -- after producing correct diffs -- is turning that granular ledger into a **single verdict view**: something a migration lead, a program sponsor, or an auditor can glance at and know, in five seconds, whether today's migration is trustworthy.

Build that view as a [Databricks AI/BI dashboard](https://docs.databricks.com/aws/en/dashboards/) backed by a query against `recon.audit_log`, structured around three components that answer three different questions:

**The big counter** answers "are we clean, right now?" -- a single large number showing the count of open (unresolved) mismatches across every reconciled table pair, refreshed on the same schedule as the underlying job. Green at zero, and nothing else on the page matters until this number is zero.

```sql
SELECT COUNT(*) AS open_mismatches
FROM recon.audit_log
WHERE run_ts = (SELECT MAX(run_ts) FROM recon.audit_log)
  AND resolved = false;
```

**The status heat map** answers "where, specifically?" -- table pairs on one axis, run dates on the other, cells colored by mismatch count. This is the view that turns "the migration has some data quality issues" into "the `orders_history` table has been failing since Tuesday, and nothing else has" -- a heat map localizes a problem in a way a single aggregate number structurally cannot.

**Drift alerts** answer "did today's clean run just become unclean?" -- not a static threshold, but a comparison against the table pair's own trailing baseline, since a table's parity check that has passed cleanly for three weeks and suddenly shows a nonzero mismatch count is a stronger signal than a table that has always run at low, tolerance-band variance. Wire this as a scheduled query alert on the audit table rather than a manual dashboard glance, so the first person to know about a regression is not the person who happened to open the tab that morning.

{: .important }
Build the dashboard against the audit table, never against a live diff computed at view time -- a dashboard that recomputes hashes on click is slow, expensive, and gives every viewer a slightly different answer depending on when they loaded the page. The audit table is the single source of truth; the dashboard only visualizes what it already recorded.

Audience matters as much as content here. A program sponsor who checks the dashboard once a week needs the big counter and nothing else -- a single glance, a single color. The migration engineer debugging a specific table pair needs the heat map and the ability to drill from a red cell straight into the underlying `diff_type` rows in the audit table, which is why the heat map should link out to a filtered query rather than stopping at the visual. Building one dashboard that serves both audiences usually means two pages, or two tabs, sharing the same underlying dataset -- an executive summary and a debugging view -- rather than one crowded page trying to be both.

It's also worth deciding upfront how the dashboard treats a mismatch that gets fixed. Because the audit table is append-only, a resolved issue doesn't disappear from history -- it should instead get a `resolved_ts` and `resolved_by` column set once someone confirms the fix, so the big counter reflects only *open* mismatches while the full historical record, including how long each issue took to close, stays available for a retrospective. That closure workflow is what turns the dashboard from a passive readout into the operational tool a migration team actually works out of every morning.

The next lecture makes the case for why this dashboard needs a fresh row in the audit table every single night, starting on day zero of the migration -- not just in the week before cutover.

<!-- prevnext:start -->

---

| [&larr; Previous: The PySpark Hash-Diff Implementation]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-pyspark-hash-diff-implementation/' | relative_url }}) | [Next: Start at Day 0, Not End of Project &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/start-at-day-0-not-end-of-project/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

