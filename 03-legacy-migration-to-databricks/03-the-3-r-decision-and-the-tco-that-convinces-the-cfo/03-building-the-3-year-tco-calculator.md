---
title: "Building the 3-Year TCO Calculator"
parent: "The 3-R Decision and the TCO That Convinces the CFO"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/building-the-3-year-tco-calculator/
read_minutes: 3
---

# Building the 3-Year TCO Calculator
{: .no_toc }

*Estimated read: 3 min*

A 3-R strategy per workload tells engineering what to build. It says nothing yet to the person who
has to approve the budget. The **three-year TCO (total cost of ownership) calculator** is the
artifact that translates the scorecard's output into the currency a CFO actually evaluates decisions
in.

Three years is the deliberate window, not an arbitrary one: shorter and you're comparing sunk legacy
license renewals against migration costs you haven't recouped yet, which makes migration look
artificially expensive; longer and compute pricing, workload growth, and platform roadmaps become too
speculative to model credibly. Three years is long enough to show the crossover point where migration
starts paying for itself, and short enough that the underlying assumptions stay defensible.

Build the model side by side -- **stay** vs **migrate** -- across every cost line a legacy estate
actually carries, not just the licensing sticker price:

| Cost line | Stay (legacy) | Migrate (Databricks) |
|---|---|---|
| Platform licensing | Oracle/Teradata core + option licenses, annual maintenance | DBU consumption (jobs/serverless compute) |
| Hardware refresh | Amortized appliance/server refresh cycle (typically every 4-5 years) | None -- no hardware to refresh |
| DBA/ops headcount | FTEs dedicated to patching, tuning, backup/recovery | Reduced -- managed service, but not zero (platform admin, FinOps) |
| Migration one-time cost | $0 | Lakebridge tooling (free/open-source) + architect and engineering time |
| Downtime/risk cost | Ongoing risk of an aging platform's failure modes | Parallel-run and cutover cost, amortized in year one |
| Storage | SAN/appliance storage, often overprovisioned | Cloud object storage, pay-for-what-you-use |

```python
years = [1, 2, 3]
stay_cost = {
    y: legacy_licensing + legacy_maintenance + (hardware_refresh_amortized if y == refresh_year else 0)
        + dba_headcount_cost + legacy_storage_cost
    for y in years
}
migrate_cost = {
    y: dbu_consumption_estimate + (migration_one_time_cost if y == 1 else 0)
        + reduced_ops_headcount_cost + cloud_storage_cost
    for y in years
}
net_savings = {y: stay_cost[y] - migrate_cost[y] for y in years}
cumulative_savings = sum(net_savings.values())
```

Two modeling details determine whether this calculator survives scrutiny:

- **Front-load the migration cost honestly.** Year one of the migrate scenario should look *more*
  expensive than staying, not less -- parallel-run compute, architect time, and Lakebridge tooling
  setup all land in year one. A TCO model where migration is cheaper in every single year reads as
  unrealistic to anyone who's lived through a real migration, and undermines trust in the rest of
  the numbers.
- **Use the scorecard's strategy mix, not a single blended rate.** Rehosted workloads have low
  migration cost and modest ongoing savings; re-architected workloads have higher migration cost and
  larger ongoing savings (they're the ones that were burning the most compute on inefficient
  patterns to begin with). Summing per-workload numbers from the scorecard, rather than applying one
  average migration cost to the whole estate, is what makes the total defensible line by line if
  someone on the finance side wants to audit it.

{: .important }
> Every number in this calculator should be either a real quote (DBU pricing, current license
> renewal cost) or a labeled assumption with its source noted next to it -- never an unlabeled
> guess. The next lecture is about presenting this model to a board; a board that finds one
> unlabeled assumption will discount every number in the deck, not just that one.

With stay-vs-migrate quantified, the next lecture is the cautionary counterexample: what happens
when a team skips the scorecard entirely and applies Rehost to every workload regardless of what the
data says.

<!-- prevnext:start -->

---

| [&larr; Previous: The Migration Assessment Scorecard]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/the-migration-assessment-scorecard/' | relative_url }}) | [Next: Anti-Pattern: Lift-and-Shift Everything &rarr;]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/anti-pattern-lift-and-shift-everything/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

