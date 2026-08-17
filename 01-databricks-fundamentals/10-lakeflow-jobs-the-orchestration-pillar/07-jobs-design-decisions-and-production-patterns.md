---
title: "Jobs Design Decisions and Production Patterns"
parent: "Lakeflow Jobs - The Orchestration Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/jobs-design-decisions-and-production-patterns/
read_minutes: 5
---

# Jobs Design Decisions and Production Patterns
{: .no_toc }

*Estimated read: 5 min*

A short closing lecture on the judgment calls worth making deliberately, before this section's
knowledge check and the move into Part 1's final destination: Part 2's full StepRight capstone.

## Run jobs as a service principal, not a person

Section 6 introduced this identity principle; restating it here because it's the single most
common production mistake with jobs specifically: a scheduled job **owned by an individual's
personal credentials** breaks the moment that person's access changes, for reasons entirely
unrelated to the job itself. Every production job in this guide runs as a dedicated **service
principal**.
{: .important }

## One job per logical pipeline, not one mega-job

Mirroring Section 9's "one pipeline per logical unit" guidance -- a single job trying to
orchestrate an entire organization's unrelated workloads makes failure diagnosis and change
management harder than several smaller, clearly-scoped jobs. Scope a job to one coherent process
(one Medallion pipeline's ingest -> transform -> quality-check -> report cycle, for instance).

## Compute sizing: jobs clusters, not all-purpose

Section 4 distinguished jobs clusters (spun up per run, torn down after, no idle cost) from
all-purpose clusters (shared, billed while running). Every scheduled job task should default to a
**jobs cluster**, not an all-purpose cluster left running -- the cost difference at scale is
substantial, and it's purely a configuration choice, not a tradeoff against functionality.

## Notification fatigue is a real design problem

A job that emails on every single run (success and failure alike) trains recipients to ignore the
emails entirely -- by the time a genuine failure happens, nobody's actually reading the alert.
Notify on failure by default; reserve success notifications for genuinely significant milestones
(a backfill completing, a first production run), not routine daily success.

## The checklist for a job you'd actually trust

1. Runs as a service principal, not a personal account.
2. Uses jobs compute, not a lingering all-purpose cluster.
3. Has a retry policy scoped to realistic transient-failure likelihood, not a blanket maximum.
4. Fails loudly and specifically -- differentiated alerts (low data vs. no data vs. quality
   violation), not one generic "something failed" email.
5. Every task it runs is idempotent, so repair runs and backfills are genuinely safe.
6. Is deployed via an asset bundle from Git, not hand-configured per environment.

## Part 1 is complete

This closes both this section and **Part 1: Databricks Fundamentals** in full -- workspace, Delta
Lake, Unity Catalog, Medallion architecture, and all three Lakeflow pillars. Everything from here
is applied directly, hands-on, in [Part 2: StepRight Capstone Project]({{ '/02-stepright-capstone-project/' | relative_url }}),
where you build a complete, tested, CI/CD-deployed data engineering project using every concept
from Part 1 together, not in isolation.

<!-- prevnext:start -->

---

| [&larr; Previous: Automating Jobs - REST API and Databricks CLI]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/automating-jobs-rest-api-and-databricks-cli/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
