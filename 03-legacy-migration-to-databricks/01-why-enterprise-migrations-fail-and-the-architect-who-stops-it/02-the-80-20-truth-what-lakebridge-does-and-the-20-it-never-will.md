---
title: "The 80/20 Truth: What Lakebridge Does and the 20% It Never Will"
parent: "Why Enterprise Migrations Fail (And the Architect Who Stops It)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-80-20-truth-what-lakebridge-does-and-the-20-it-never-will/
read_minutes: 3
---

# The 80/20 Truth: What Lakebridge Does and the 20% It Never Will
{: .no_toc }

*Estimated read: 3 min*

**[Lakebridge](https://databrickslabs.github.io/lakebridge/docs/overview/)** is Databricks' free,
open-source toolkit for exactly the mechanical work described in the previous lecture: it covers the
full arc from surveying your existing SQL estate through translating it to run on Databricks SQL,
to validating that the migrated data reconciles with the source. It's built around three phases that
map directly onto the operating model you'll use throughout this part:

| Phase | What it does | Legacy-world equivalent |
|---|---|---|
| **Assessment** | Scans your source SQL workloads and estimates migration complexity, effort, and TCO impact | The manual spreadsheet-and-guesswork sizing exercise you used to do before a warehouse consolidation |
| **Transpilation** | Converts SQL and orchestration logic from source dialects (T-SQL, Snowflake, Oracle, Teradata, Redshift, BigQuery) into ANSI SQL that runs on Databricks, using one of three transpiler engines (BladeBridge, Morpheus, Switch) | A DDL/DML converter, but one that also understands dialect-specific functions and join syntax |
| **Reconciliation** | Compares migrated datasets in Databricks against the source system to validate the transfer | The row-count-and-spot-check step you already do after any ETL cutover, formalized and automated |

More recently, Databricks added the **[Lakebridge Agentic
Converter](https://docs.databricks.com/aws/en/migration/lakebridge-agentic-converter)**, which
organizes conversion work into projects: it analyzes source scripts, generates the ANSI SQL
equivalent, validates the result, and iteratively retries sections that fail -- with a side-by-side
review UI and the ability to encode your organization's own business rules as reusable "skills" that
apply automatically across conversions.

That's the 80%, and it's a genuinely large 80%. Straightforward `CREATE TABLE` and `CREATE VIEW`
DDL, standard `SELECT` transformations, common built-in function calls, and a large share of batch
ETL logic translate with high confidence and minimal manual rework. For a legacy engineer used to
hand-writing every conversion in a migration project, this alone can cut months off a timeline.

Here's the 20% no transpiler closes, regardless of how good the model behind it gets:

- **Procedural logic with side effects.** A stored procedure that opens a cursor, mutates a
  package-level global variable, and conditionally calls three other procedures based on runtime
  state doesn't have a mechanical SQL-to-SQL translation -- it needs to be *decomposed* into
  set-based DataFrame operations, which is a design decision, not a syntax conversion. You'll do
  this work directly in the Procedure Autopsy section later in this part.
- **Business intent that isn't in the code.** Why does this report exclude rows where
  `status_cd = 'X'`? Sometimes it's a documented business rule; sometimes it's a workaround for a
  data quality bug from 2014 that nobody remembers, and blindly porting it forward preserves a bug
  as a feature.
- **Semantic equivalence, not just syntactic equivalence.** A transpiler can produce SQL that
  compiles and runs. Whether it produces the *same numbers* as the source system -- to the same
  rounding behavior, the same null-handling, the same sort-order-dependent row selection -- is a
  reconciliation question, and reconciliation requires a human to define what "close enough"
  means for a given metric.
- **Judgment calls the 3-R decision requires.** Whether a workload should be rehosted, re-platformed,
  or re-architected is a cost/risk tradeoff informed by business priorities Lakebridge has no
  visibility into.

Treat Lakebridge as a very capable junior engineer who can turn around a first-pass translation of
almost anything overnight, but who cannot be the one who signs off that the translation is *correct*.
That sign-off is the migration architect's job, and it's the one 20% you can't automate away.

<!-- prevnext:start -->

---

| [&larr; Previous: The Migration Graveyard: Why 1 in 3 EDW Migrations Stall]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-migration-graveyard-why-1-in-3-edw-migrations-stall/' | relative_url }}) | [Next: The Migration Architect's Operating Model &rarr;]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-migration-architects-operating-model/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

