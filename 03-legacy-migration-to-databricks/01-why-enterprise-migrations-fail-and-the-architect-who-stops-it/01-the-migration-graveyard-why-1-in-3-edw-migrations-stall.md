---
title: "The Migration Graveyard: Why 1 in 3 EDW Migrations Stall"
parent: "Why Enterprise Migrations Fail (And the Architect Who Stops It)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-migration-graveyard-why-1-in-3-edw-migrations-stall/
read_minutes: 3
---

# The Migration Graveyard: Why 1 in 3 EDW Migrations Stall
{: .no_toc }

*Estimated read: 3 min*

Industry studies on large-scale data platform migrations put the failure rate somewhere between one
in three and one in two: projects that blow their budget by multiples, slip past their original
go-live date by a year or more, or get quietly shelved after the steering committee loses confidence.
If you've spent years running a legacy **EDW** (enterprise data warehouse) -- Oracle, Teradata, SQL
Server -- you've probably watched one of these die, or at least heard the post-mortem secondhand.
The pattern repeats often enough that it's worth naming precisely, because the failure mode is
rarely the target platform. It's how the project was scoped.

Three causes account for most of the graveyard:

1. **The estate was never really profiled.** Someone counted tables and estimated storage, called
   it an assessment, and moved straight to a migration timeline. Nobody ran the equivalent of an
   Oracle AWR report against the *procedural* layer -- the stored procedures, triggers, and
   scheduler jobs that actually encode the business logic. You'll build that workload inventory
   properly in the next section; skipping it is the single most common root cause of a stalled
   migration.
2. **Reconciliation was an afterthought.** Teams migrate schema and data, run a spot-check row
   count, declare victory, and only discover in month four of the parallel run that a `SUM()` over
   a `NUMBER(38,10)` column in Oracle doesn't match a Spark `DecimalType` sum to the last two digits
   of precision -- or that a report's row ordering silently depended on Oracle's default sort
   behavior. By then the business has lost trust in the new platform, and trust, once lost mid-
   migration, is expensive to rebuild.
3. **Everything got lift-and-shifted.** Rehosting every workload as-is is sometimes the right call
   for a workload under deadline pressure, but applied to *everything* it means you pay to
   re-implement inefficient legacy patterns -- cursor-driven row-by-row procedures, nightly full
   reloads -- on a platform that was built for a fundamentally different execution model. You get a
   working migration and none of the cost or performance benefit that justified doing it.

None of these are technology problems. Lakebridge and the rest of the Databricks migration tooling
solve the mechanical parts -- DDL translation, SQL transpilation, orchestration conversion -- well.
They do not solve scoping, and they do not tell you which 20% of your procedures encode the business
rules that actually matter. That gap is exactly what the rest of this part, and the role of a
**migration architect**, exists to close.

{: .important }
> The projects that succeed treat migration as a *business-logic extraction* project that happens to
> move data, not a data-movement project that happens to touch some logic. Everything in this part
> -- the autopsy, the 3-R decision, reconciliation, the parallel run -- follows from taking that
> framing seriously from day one.

The next lecture looks at exactly where that line falls: what Lakebridge automates reliably, and the
20% of any real migration that still requires a human who understands both the source system and the
lakehouse.

<!-- prevnext:start -->

---

| [&larr; Previous: Why Enterprise Migrations Fail (And the Architect Who Stops It)]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/' | relative_url }}) | [Next: The 80/20 Truth: What Lakebridge Does and the 20% It Never Will &rarr;]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-80-20-truth-what-lakebridge-does-and-the-20-it-never-will/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

