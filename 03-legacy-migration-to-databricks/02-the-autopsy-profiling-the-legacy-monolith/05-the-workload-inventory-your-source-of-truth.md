---
title: "The Workload Inventory: Your Source of Truth"
parent: "The Autopsy: Profiling the Legacy Monolith"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/the-workload-inventory-your-source-of-truth/
read_minutes: 4
---

# The Workload Inventory: Your Source of Truth
{: .no_toc }

*Estimated read: 4 min*

Everything the last four lectures produced -- bottleneck rankings, the dependency graph, the heat
map -- is diagnostic raw material. The **workload inventory** is where it becomes a decision-making
artifact: one row per object (table, view, procedure, job) with a **defensible verdict**, and every
verdict traceable back to a rerunnable query rather than a memory of a conversation in a hallway.

A workload inventory that will actually survive contact with a steering committee needs these
columns at minimum:

| Column | Source | Purpose |
|---|---|---|
| `object_name`, `object_type` | Catalog metadata | Identity |
| `row_count`, `storage_gb` | Catalog metadata | The visible 20% -- size, not priority |
| `access_frequency_30d` | AWR / DBQL / Query Store | How hot is this, right now |
| `dependent_object_count` | Dependency graph | How much breaks if this changes |
| `avg_cpu_or_io_cost` | AWR / DBQL / Query Store | Migration-priority ranking signal |
| `verdict` | Architect judgment | **Lift**, **Redesign**, or **Retire** |
| `verdict_rationale` | Architect judgment | One sentence -- why this verdict, not another |
| `source_query_id` | Link back to the query that produced the row | Reproducibility |

The three verdicts map directly onto the 3-R decision the next section formalizes: **Lift** is a
strong candidate for rehosting largely as-is, **Redesign** signals re-platform or re-architect work,
and **Retire** means exactly what it says -- an orphaned procedure or a table with a year of zero
access is scope you remove, not scope you migrate. Assigning these three verdicts *before* the next
section's scorecard exists isn't circular -- it's a first pass that the 3-R scorecard will later
formalize and sometimes overturn, and having an initial hypothesis per object makes the scorecard
conversation faster, not slower.

The `source_query_id` column is the detail most inventories skip, and the one that matters most when
someone on the steering committee asks "how do you know this table only gets touched twice a
month?" six weeks after you compiled the sheet. Every cell that isn't a raw catalog fact should trace
back to a specific, rerunnable query against AWR, DBQL, Query Store, or the dependency graph query
from the previous lecture -- not a number someone remembers pulling once. This is what makes the
inventory a **source of truth** rather than a snapshot opinion: anyone on the team, including someone
who joins the project in month four, can rerun the underlying query and confirm the verdict still
holds, or flag that it's drifted.

{: .important }
> An inventory without traceable sources decays into an unverifiable spreadsheet within weeks --
> and an unverifiable spreadsheet is exactly the kind of artifact a stalled migration gets blamed on
> in the post-mortem. Build traceability in from row one; retrofitting it after the fact means
> re-running every query anyway, with none of the goodwill of having done it right the first time.

This inventory is the deliverable this section has been building toward, and it's the input every
subsequent section in this part consumes: the 3-R decision reads its verdicts, the TCO calculator
reads its cost signals, and the sequencing plan in the migration playbook reads its dependency
counts. Get the autopsy right here, and the rest of the migration is executing against a known map
instead of discovering the territory as you go.

<!-- prevnext:start -->

---

| [&larr; Previous: Stored-Procedure Dependency Graphs and Table Heat Maps]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/stored-procedure-dependency-graphs-and-table-heat-maps/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

