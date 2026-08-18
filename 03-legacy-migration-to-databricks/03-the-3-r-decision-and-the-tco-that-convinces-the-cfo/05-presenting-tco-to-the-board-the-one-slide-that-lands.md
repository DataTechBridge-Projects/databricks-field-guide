---
title: "Presenting TCO to the Board: The One Slide That Lands"
parent: "The 3-R Decision and the TCO That Convinces the CFO"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/presenting-tco-to-the-board-the-one-slide-that-lands/
read_minutes: 2
---

# Presenting TCO to the Board: The One Slide That Lands
{: .no_toc }

*Estimated read: 2 min*

The TCO calculator from earlier in this section can easily run to forty rows once every cost line
and every workload's strategy is accounted for. A board or CFO will not read forty rows. They will
read one slide, ask two questions about it, and make a decision based on how well those two
questions get answered. This lecture is about building that slide from the calculator you already
have -- not a new artifact, a distillation of the one you built.

**The slide has exactly three elements, in this order:**

1. **The headline number.** Three-year net savings, as a single dollar figure, stated plainly:
   "Migrating saves $X over three years versus staying on the current platform." Not a range, not a
   percentage buried in a chart -- the number the calculator's `cumulative_savings` produced,
   rounded to something readable.
2. **A simple year-by-year bar chart.** Stay cost vs. migrate cost, for years one through three,
   directly from the calculator's `stay_cost` and `migrate_cost` dictionaries. This is where the
   honest front-loading from the previous TCO lecture pays off visually: year one shows migration
   costing *more*, and years two and three show the crossover -- which reads as credible precisely
   because it isn't a straight line claiming savings from day one.
3. **One line of narrative underneath**, naming the strategy mix in plain terms: "X% of workloads
   rehost as-is; Y% are re-platformed; Z% -- the highest-value, most frequently changed -- are
   re-architected for real performance and cost gains." This preempts the most common board question
   ("are we just moving the same problems to a new platform?") before anyone has to ask it.

{: .important }
> Every number on this slide must trace back to a specific cell in the underlying TCO calculator --
> the same traceability discipline from the workload inventory applies here. The first hard question
> from a sharp CFO is usually "where does that number come from," and "let me pull up the model" is
> a far stronger answer than a paraphrase from memory.

What doesn't belong on this slide: technology detail (nobody on a board needs to hear "Liquid
Clustering" to approve a budget), a risk-free framing (the year-one dip is the honest risk, and
naming it builds more trust than hiding it), or a multi-year roadmap with dozens of milestones --
that belongs in the appendix, available if asked for, not on the slide that has to land in the thirty
seconds of attention a board actually gives a budget ask.

Keep the underlying model editable and attached as backup, not merged into the slide itself. The
slide is the pitch; the calculator is the evidence, and you want both available but visually
separate, so a follow-up question can be answered by opening the model instead of scrolling past the
narrative slide to find it.

With the 3-R decision made, the scorecard justified, the TCO modeled, and the board narrative built,
this section's toolkit is complete. The next section shifts from justification to execution: what
Lakehouse Federation lets you do *before* a single table physically moves.

<!-- prevnext:start -->

---

| [&larr; Previous: Anti-Pattern: Lift-and-Shift Everything]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/anti-pattern-lift-and-shift-everything/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

