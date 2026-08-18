---
title: "Check Your Knowledge"
parent: "The 3-R Decision and the TCO That Convinces the CFO"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you've learned from this section before moving on to Lakehouse Federation.

1. Which 3-R strategy involves moving a workload with minimal change to its logical schema or procedural structure?
   A. Re-architect
   B. Rehost
   C. Re-platform
   D. Retire

2. According to the decision tree, a workload with high business value and deep PL/SQL procedural logic should generally be:
   A. Rehosted
   B. Retired
   C. Re-architected
   D. Left on the legacy platform indefinitely

3. In the migration assessment scorecard, which three dimensions are combined into the `re_architect_score`?
   A. Technical complexity, data volume, dependency fanout
   B. Business value, change frequency, PL/SQL depth
   C. Data volume, dependency fanout, technical complexity
   D. PL/SQL depth, data volume, business value

4. Why does the scorecard weight business value and change frequency more heavily than raw technical complexity when recommending a strategy?
   A. Complexity is irrelevant to migration effort
   B. Value and change frequency determine whether the re-architecture investment actually pays off, while complexity mainly affects effort estimation
   C. Technical complexity cannot be measured objectively
   D. Data volume always outweighs the other dimensions

5. Why is a three-year window used for the TCO calculator rather than one year or ten years?
   A. It matches Databricks' standard contract length
   B. It's long enough to show the savings crossover point while staying short enough for assumptions to remain credible
   C. Oracle license renewals are always three years
   D. It is required by GAAP accounting standards

6. What does "front-loading the migration cost honestly" mean in the TCO calculator?
   A. Showing year one of the migrate scenario as cheaper than staying
   B. Showing year one of the migrate scenario as more expensive than staying, reflecting real parallel-run and setup costs
   C. Deferring all migration costs to year three
   D. Excluding migration costs from the model entirely

7. What is the core problem with the "lift-and-shift everything" anti-pattern, as described in this section?
   A. Lakebridge cannot transpile any SQL without a full re-architecture
   B. It appears successful at go-live but leaves inefficient legacy patterns and consumes compute wastefully, deferring the promised re-architecture ROI
   C. It is technically impossible to rehost more than a few tables at once
   D. It always causes the migration to miss its go-live date

8. Per the anti-pattern lecture, what is the recommended response if schedule pressure is genuinely forcing an across-the-board rehost?
   A. Proceed silently and let the TCO case go stale
   B. Cancel the migration entirely
   C. Surface the tradeoff explicitly to the stakeholders who approved the TCO case
   D. Re-architect everything regardless of the schedule

9. What are the three required elements of the one-slide board presentation described in this section?
   A. A risk register, a Gantt chart, and a budget line item
   B. The headline savings number, a year-by-year stay-vs-migrate bar chart, and one line of narrative on strategy mix
   C. A full architecture diagram, a security review, and a compliance checklist
   D. A list of every migrated table and its row count

10. Why should the year-one bar in the TCO chart show migration costing more than staying?
    A. It's a formatting requirement of most presentation software
    B. It reflects the honest front-loaded migration cost, which makes the crossover in later years read as credible rather than suspicious
    C. Year one costs are always excluded from board presentations
    D. Databricks requires a minimum first-year spend commitment

## Answer Key

1. **B** -- Rehost means moving the workload with minimal change to schema or procedural structure.
2. **C** -- High value combined with deep procedural logic is the branch that leads to re-architect in the decision tree.
3. **B** -- `re_architect_score` sums business value, change frequency, and PL/SQL depth (max 15).
4. **B** -- Value and change frequency determine ROI on redesign effort; complexity and volume mainly inform effort estimation, not whether redesign pays off.
5. **B** -- Three years is long enough to show the stay-vs-migrate crossover while keeping cost and growth assumptions credible.
6. **B** -- Honest front-loading shows migration as more expensive in year one, reflecting real parallel-run and setup costs, not an unrealistic immediate savings claim.
7. **B** -- Rehosting everything looks successful at go-live but leaves inefficient patterns in place, inflates compute cost, and defers the re-architecture ROI the TCO case promised.
8. **C** -- The tradeoff (faster go-live vs. deferred re-architecture cost) should be surfaced explicitly to the stakeholders who approved the original TCO case, not absorbed silently.
9. **B** -- The slide needs the headline savings number, a year-by-year bar chart, and one line naming the rehost/re-platform/re-architect mix.
10. **B** -- The honest year-one dip, visible on the chart, is what makes the later-year crossover credible rather than an unrealistic straight-line savings claim.

<!-- prevnext:start -->

---

| [&larr; Previous: Presenting TCO to the Board: The One Slide That Lands]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/presenting-tco-to-the-board-the-one-slide-that-lands/' | relative_url }}) | [Next: Lakehouse Federation: Migrate Without Migrating &rarr;]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

