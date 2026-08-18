---
title: "How This Course Works and Your Downloadable Toolkit"
parent: "Why Enterprise Migrations Fail (And the Architect Who Stops It)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/how-this-course-works-and-your-downloadable-toolkit/
read_minutes: 3
---

# How This Course Works and Your Downloadable Toolkit
{: .no_toc }

*Estimated read: 3 min*

This part is structured so that by the time you reach the capstone war room, you're not reasoning
about a migration in the abstract -- you're holding four artifacts you built section by section, the
same way you'd walk into a real engagement with a toolkit rather than a blank page.

**The TCO calculator.** Built in *The 3-R Decision and the TCO That Convinces the CFO*, this is a
three-year total-cost-of-ownership model that compares the fully loaded cost of staying on the
legacy platform (licensing, hardware refresh cycles, the DBA headcount required to keep a
twenty-year-old system patched) against the Databricks-side cost across rehost, re-platform, and
re-architect scenarios. You'll use it to build the one-slide summary that actually lands with a
board, not a forty-tab spreadsheet nobody reads past slide two.

**The reconciliation script library.** Built across *The Reconciliation Stack* and *Building the
Reconciliation Engine*, this is a set of PySpark jobs implementing the five-layer reconciliation
stack -- count, sum, checksum, hash, and semantic parity -- plus the dashboards that make parity
visible to a business stakeholder who doesn't want to read a Spark job to trust a cutover.

**The ABAC tag taxonomy.** Built in *ABAC, Masking, and Cross-Engine Governance*, this is the tag
schema and masking policy pack that replaces a legacy role hierarchy of five hundred Oracle roles
with a governed-tags model in Unity Catalog -- schema-level attribute tagging instead of a role
explosion, so access policy scales with your data model instead of against it.

**The go/no-go decision matrix.** Built in *The Migration Playbook* and used for real in the
capstone, this is the structured checklist -- reconciliation status, rollback readiness, business
sign-off, compute headroom -- that turns a cutover decision from a gut call under pressure into a
defensible, auditable one.

Each artifact is introduced where it's built, with working code and templates you can adapt directly
to your own migration -- this isn't a toy example you'll discard after the section ends. The
sandbox you'll use throughout mirrors StepRight's structure from Part 2: a bronze/silver/gold Unity
Catalog setup, except now seeded with data and DDL that mimics a legacy Oracle-style schema (surrogate
keys via sequences, PL/SQL-flavored naming, denormalized reporting tables) so the translation and
reconciliation exercises are grounded in something that looks like what you'll actually inherit on
the job.

{: .important }
> Treat every code sample and template in this part as a starting point, not a finished tool -- the
> actual column names, thresholds, and tag taxonomy in your migration will differ from StepRight's.
> What should transfer directly is the *structure*: which five layers a reconciliation job needs,
> which fields a go/no-go matrix has to cover, which cost lines a TCO calculator can't afford to
> omit.

With the operating model and the toolkit both in view, the next section starts the work for real:
profiling the legacy monolith you've inherited, starting with the question every migration skips at
its own peril -- what's actually in this system, and what does it actually cost to run?

<!-- prevnext:start -->

---

| [&larr; Previous: The Migration Architect's Operating Model]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-migration-architects-operating-model/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

