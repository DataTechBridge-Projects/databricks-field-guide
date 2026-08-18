---
title: "Anti-Pattern: Lift-and-Shift Everything"
parent: "The 3-R Decision and the TCO That Convinces the CFO"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/anti-pattern-lift-and-shift-everything/
read_minutes: 2
---

# Anti-Pattern: Lift-and-Shift Everything
{: .no_toc }

*Estimated read: 2 min*

Rehosting isn't wrong -- it's the correct call for a meaningful share of any real estate, exactly as
the decision tree earlier in this section showed. The anti-pattern is applying it to *everything*,
usually because it's the fastest path to a demo-able migration and the schedule pressure is real.
It's worth naming explicitly because it's the single most common way a technically successful
migration still ends up a strategic failure.

Here's how it plays out. A team under deadline pressure skips the scorecard, translates every table
and procedure as close to line-for-line as Lakebridge and manual effort allow, and hits go-live on
schedule. Six months later, three things are true simultaneously:

- **The cursor-driven procedures that were slow on Oracle are still slow on Databricks**, because
  row-by-row processing logic doesn't get faster by changing which engine runs it -- it needed to
  become a set-based DataFrame operation, and a straight rehost never touched that.
- **The compute bill is higher than projected**, because inefficient legacy patterns -- serial
  execution, full-table nightly reloads where an incremental `MERGE` would do -- consume DBUs just
  as wastefully as they consumed CPU cycles on-prem. Migration didn't fix the underlying design; it
  just moved the same design onto metered infrastructure.
- **None of the re-architecture ROI the original TCO case promised shows up**, because that ROI was
  modeled assuming the high-value, high-change-frequency workloads would get re-architected --
  and they didn't, because everything got the same rehost treatment under time pressure.

{: .important }
> A rehost-everything migration is not a failed migration by the metric most steering committees
> track (did we cut over on schedule) -- which is exactly what makes it dangerous. It looks
> successful at go-live and only reveals its cost twelve months later, in a compute bill nobody can
> explain and a re-architecture backlog nobody budgeted for.

The fix isn't heroics under deadline pressure -- it's protecting the scorecard step earlier in the
schedule, not cutting it when the timeline gets tight. If schedule pressure is genuinely forcing an
across-the-board rehost, that's a decision worth surfacing explicitly to the same stakeholders who
approved the TCO case, with the tradeoff named: faster go-live, deferred re-architecture cost, and a
compute bill that won't match the original projection until that deferred work gets done. Silently
defaulting to rehost-everything and letting the TCO case go stale is the actual anti-pattern here --
not rehosting itself.

The next lecture closes this section with the flip side of this problem: how to present the TCO case
-- including an honest accounting of what gets rehosted versus re-architected -- to a board in a way
that survives scrutiny instead of collapsing under the first hard question.

<!-- prevnext:start -->

---

| [&larr; Previous: Building the 3-Year TCO Calculator]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/building-the-3-year-tco-calculator/' | relative_url }}) | [Next: Presenting TCO to the Board: The One Slide That Lands &rarr;]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/presenting-tco-to-the-board-the-one-slide-that-lands/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

