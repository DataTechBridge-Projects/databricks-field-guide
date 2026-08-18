---
title: "Start at Day 0, Not End of Project"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/start-at-day-0-not-end-of-project/
read_minutes: 3
---

# Start at Day 0, Not End of Project
{: .no_toc }

*Estimated read: 3 min*

The most common mistake in a migration schedule is treating reconciliation as a phase instead of a habit -- something scheduled for the two weeks before cutover, after every table has already been migrated and every PL/SQL procedure already translated. By the time that phase starts, every defect it finds has been sitting in the codebase for weeks or months, and the team fixing it has to reconstruct context on code nobody has touched since the initial translation pass.

This is a **defect cost curve** problem, the same one that governs every other flavor of software quality: a bug caught the day it's introduced costs a few minutes to fix, because the engineer who wrote it still has the whole context loaded and the change is small and isolated. The same bug caught eight weeks later, discovered during a reconciliation phase against a codebase that's since grown three more tables downstream of it, costs an investigation, a re-review of everything built on top of the bad assumption, and a scramble against a cutover date that was set before anyone knew this rework was coming. The reconciliation engine built in this section doesn't change the *shape* of that curve -- it changes *when* on the curve each bug gets caught, and the entire value of building the engine early is moving that catch point as far left as possible.

The mechanism is a **Lakeflow Job** that runs the hash-diff engine from earlier in this section nightly, against whatever tables have been migrated so far -- even if that's three tables in week one of a nine-month program. See the [Lakeflow Jobs documentation](https://docs.databricks.com/aws/en/jobs/) for the orchestration primitives (tasks, schedules, retries) this job runs on. Each night's run appends to the same Delta audit table, so the dashboard from the previous lecture is live and meaningful from day zero, not something that gets switched on in month eight.

Running reconciliation this early does something beyond catching bugs sooner: it validates the reconciliation engine itself, against real data, while there's still time to fix a broken hash function or a mis-cast join key. A team that builds its parity-checking logic in week one and runs it nightly for months has enormous confidence in it by cutover, because it has already caught -- and the team has already fixed -- dozens of small drift issues along the way. A team that builds the same logic in week thirty-four, under deadline pressure, is debugging both the migration *and* the tool meant to verify the migration, simultaneously, with no time to spare for either.

{: .important }
Nightly reconciliation from day zero also reframes what "done" means for an individual table migration -- a table isn't finished when the PySpark job runs without error, it's finished when it's shown up clean in the reconciliation dashboard for several consecutive nights. That's a stronger, more honest completion criterion than "the code compiled and I eyeballed a sample."

The final lecture in this section turns this nightly job from a single table-pair script into a portable library that can be pointed at any table pair in the migration inventory with nothing but a config change.

<!-- prevnext:start -->

---

| [&larr; Previous: Reconciliation Dashboards: Making Parity Visible]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/reconciliation-dashboards-making-parity-visible/' | relative_url }}) | [Next: The Reconciliation Script Library &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-reconciliation-script-library/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

