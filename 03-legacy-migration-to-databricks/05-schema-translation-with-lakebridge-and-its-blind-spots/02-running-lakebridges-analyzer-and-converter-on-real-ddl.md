---
title: "Running Lakebridge's Analyzer and Converter on Real DDL"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/running-lakebridges-analyzer-and-converter-on-real-ddl/
read_minutes: 3
---

# Running Lakebridge's Analyzer and Converter on Real DDL
{: .no_toc }

*Estimated read: 3 min*

The type-mapping matrix from the previous lecture is the theory. This lecture is the hands-on
version: pointing Lakebridge's [analyzer and
transpiler](https://databrickslabs.github.io/lakebridge/docs/transpile/) at a real export of source
DDL and reading what comes back.

## Step 1: Analyze before you convert

Before translating anything, run the **analyzer** against an exported directory of source DDL and
SQL. It doesn't convert a single statement -- it inventories the estate and estimates effort:

```bash
databricks labs lakebridge analyze \
  --source-directory ./exports/oracle_ddl \
  --source-tech oracle \
  --report-file ./reports/oracle_assessment.xlsx \
  --generate-json true
```

The output is a workbook (plus an optional JSON file for programmatic use) covering the object
inventory -- tables, views, and procedures found -- a complexity/effort score per object, and the
cross-object dependencies between them. Run this first for the same reason you ran the workload
inventory before the 3-R decision: you want a complete map of what you're converting *before* you
start converting it, not a surprise mid-migration.

## Step 2: Convert with the transpiler

Once you know what you're working with, run the **converter** against the same source directory:

```bash
databricks labs lakebridge transpile \
  --transpiler-config-path ./lakebridge_config.json \
  --input-source ./exports/oracle_ddl \
  --source-dialect oracle \
  --output-folder ./converted/databricks_sql
```

Lakebridge ships three transpiler engines under the hood -- **BladeBridge** (broadest dialect and
ETL/orchestration coverage), **Morpheus** (newer, with dbt support), and **Switch** (LLM-powered,
for cases the deterministic engines can't handle) -- selected via the transpiler config rather than
chosen per-statement. `install-transpile` walks you through picking one and setting the source
dialect and output folder once, so day-to-day `transpile` runs don't need every flag repeated.

Run the converter against DDL specifically first, before pointing it at the larger and messier
world of `SELECT` transformations and stored-procedure bodies. A schema that's already reviewed and
locked gives every downstream conversion step -- views, procedures, pipeline code -- a stable target
to translate against, instead of chasing a moving definition of what the target table even looks
like.

## Step 3: Read the converted DDL like a code review, not a diff approval

The output folder now holds `CREATE TABLE` statements written in Databricks SQL. The temptation at
this point is to skim them, confirm they compile, and move on -- resist it. Two things to check on
every converted object before it goes anywhere near a real catalog:

- **Every type mapping decision the converter made silently.** Cross-reference against the matrix
  from the previous lecture, especially any unconstrained `NUMBER` or Oracle `DATE` columns --
  those are exactly the cases where the "correct" mapping is ambiguous and the converter's default
  may not be your organization's default.
- **Constraints and defaults that don't survive translation.** `CHECK` constraints referencing
  source-specific functions, `DEFAULT` expressions calling Oracle sequences or Teradata
  `identity`-style columns, and foreign keys pointing at objects outside the exported scope are the
  usual casualties -- the converter either drops them, comments them out, or translates them into
  something that parses but doesn't mean the same thing.

The next two lectures dig into exactly those two failure modes in depth: silent precision loss in
numeric types, and semantic drift in generated DDL more broadly.

<!-- prevnext:start -->

---

| [&larr; Previous: The Data Type Mapping Matrix]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/the-data-type-mapping-matrix/' | relative_url }}) | [Next: The Silent Precision-Loss Bug &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/the-silent-precision-loss-bug/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

