---
title: "The Migration Architect's Operating Model"
parent: "Why Enterprise Migrations Fail (And the Architect Who Stops It)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-migration-architects-operating-model/
read_minutes: 3
---

# The Migration Architect's Operating Model
{: .no_toc }

*Estimated read: 3 min*

Every section in this part maps onto one stage of a six-stage operating model. Think of it as the
runbook a **migration architect** carries into an engagement -- not a rigid waterfall (stages
overlap and loop back constantly), but a checklist of the work that has to happen *somewhere* before
you cut over, in roughly this order:

1. **Assess.** Profile the legacy estate: read the Oracle AWR reports or Teradata DBQL logs to find
   real bottlenecks, not assumed ones; build stored-procedure dependency graphs and table heat maps;
   produce a workload inventory that becomes your source of truth for scope. Covered in *The
   Autopsy: Profiling the Legacy Monolith*.
2. **Decide.** For each workload, choose rehost, re-platform, or re-architect based on a scorecard,
   not a gut call, and build the three-year TCO case that gets a CFO to sign off. Covered in *The
   3-R Decision and the TCO That Convinces the CFO*.
3. **Translate.** Convert schema with Lakebridge, decompose stored procedures into their business
   logic, and translate procedural patterns (cursors, triggers, temp tables, `MERGE`) into their
   lakehouse-native equivalents -- with an LLM-assisted workflow that still runs through a human
   gate. Covered across the schema translation, procedure autopsy, pattern translation, and
   AI-assisted migration sections.
4. **Ingest.** Stand up the CDC and batch ingestion pipelines -- Lakeflow Connect, Auto Loader, a
   partner CDC tool -- that will feed the new platform going forward, with data contracts enforced
   at the bronze layer. Covered in *The Ingestion Decision Tree* and the CDC section that follows it.
5. **Reconcile and cut over.** Build the reconciliation stack that proves semantic parity between
   old and new, run a parallel run long enough to build trust, and execute a cutover with a rollback
   window you've actually tested. Covered across three sections culminating in *The Parallel Run and
   Zero-Downtime Cutover*.
6. **Govern and optimize.** Migrate privileges into Unity Catalog, replace role-explosion with ABAC
   tags, and run the FinOps discipline -- system tables, chargeback, compute arbitrage -- that keeps
   the new platform's bill defensible. Covered in the governance and cost sections that close out
   this part before the capstone.

{: .important }
> Stages 3 through 6 are not strictly sequential in a real engagement -- you'll often be
> reconciling one workload while still translating another. What *is* sequential is the discipline:
> you cannot skip straight to cutover on a workload that hasn't been reconciled, no matter how much
> schedule pressure is on it. Every graveyard story in the first lecture of this section traces back
> to a stage that got skipped, not one that ran out of order.

This model is also how the rest of this part is organized -- each numbered stage above corresponds
to one or more sections ahead, so if you ever lose the thread of why a given lecture matters, this
is the map to come back to. The next lecture walks through the toolkit you'll build hands-on as you
move through it: a TCO calculator, a reconciliation script library, an ABAC tag taxonomy, and the
go/no-go decision matrix you'll use in the capstone war room.

<!-- prevnext:start -->

---

| [&larr; Previous: The 80/20 Truth: What Lakebridge Does and the 20% It Never Will]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-80-20-truth-what-lakebridge-does-and-the-20-it-never-will/' | relative_url }}) | [Next: How This Course Works and Your Downloadable Toolkit &rarr;]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/how-this-course-works-and-your-downloadable-toolkit/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

