---
title: "The Silent Precision-Loss Bug"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/the-silent-precision-loss-bug/
read_minutes: 3
---

# The Silent Precision-Loss Bug
{: .no_toc }

*Estimated read: 3 min*

Of every category of migration bug covered in this part, this is the one most likely to reach
production undetected, because it doesn't throw an error. The converted DDL compiles. The load
succeeds. The row counts reconcile. And the numbers are still, quietly, wrong.

## Where it comes from

The root cause is the ambiguity flagged in the type-mapping matrix two lectures back: Oracle's
`NUMBER` type with no declared precision or scale. Oracle allows it because the engine stores
numeric values in a variable-length internal format and only needs precision/scale when you ask it
to round or format. Delta has no equivalent -- every `DECIMAL` column has a fixed, declared
precision and scale, and every `DOUBLE` column is IEEE 754 floating point. Something has to give
when an unconstrained `NUMBER` crosses that boundary, and what gives is precision.

Two failure patterns show up in practice:

- **Widening to `DOUBLE`.** A converter (or a human doing a fast manual translation) maps
  unconstrained `NUMBER` to `DOUBLE` because it's the path of least resistance -- no precision or
  scale to guess at. The problem: `DOUBLE` is binary floating point, and values like `0.1` have no
  exact binary representation. A financial aggregate that summed cleanly in Oracle's decimal
  arithmetic accumulates rounding error across millions of rows in Databricks, and the drift is
  small enough per-row to survive a spot check but large enough in aggregate to fail a finance
  team's reconciliation.
- **Truncating to an undersized `DECIMAL`.** The opposite mistake: mapping to `DECIMAL(p,s)` with
  `p`/`s` too small for the actual data. Oracle silently accepted a `NUMBER` value with 12
  significant digits for years because nothing constrained it; the moment that value lands in a
  `DECIMAL(10,2)` column, Databricks either rejects the row (if type coercion is strict) or rounds
  it (if it isn't) -- and a rounded financial value is worse than a rejected one, because it doesn't
  raise an alarm.

## How to catch it before it ships

```sql
-- Profile the actual precision/scale in use, not the declared type,
-- against a representative sample or the full column if the table is small enough.
SELECT
  MAX(LENGTH(CAST(amount AS STRING)) - INSTR(CAST(amount AS STRING), '.')) AS max_scale_observed,
  MAX(LENGTH(REPLACE(CAST(amount AS STRING), '.', ''))) AS max_precision_observed
FROM legacy_export.gl_transactions;
```

Run a query like this against a representative export from the *source* system before finalizing
the target `DECIMAL(p,s)`, then set precision and scale a few digits wider than the observed
maximum -- not exactly at it, since the sample you profiled may not contain the widest value the
table has ever held.

{: .important }
> Never let an unconstrained numeric type resolve to `DOUBLE` for anything that will be summed,
> compared for equality, or reported to a human as a dollar figure. Use `DECIMAL` with an explicit,
> deliberately-chosen precision and scale every time, even when it means going back to the source
> DBA to ask what range of values a column has actually held.

This is exactly the kind of defect that count-based reconciliation misses and hash- or
sum-based reconciliation catches -- a preview of why the reconciliation stack later in this part
uses multiple validation strategies rather than relying on row counts alone.

<!-- prevnext:start -->

---

| [&larr; Previous: Running Lakebridge's Analyzer and Converter on Real DDL]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/running-lakebridges-analyzer-and-converter-on-real-ddl/' | relative_url }}) | [Next: Auditing Generated DDL for Semantic Drift &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/auditing-generated-ddl-for-semantic-drift/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

