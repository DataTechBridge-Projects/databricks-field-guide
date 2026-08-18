---
title: "Migrating the Hard Ones: 200-Column Teradata Tables and Geospatial"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/migrating-the-hard-ones-200-column-teradata-tables-and-geospatial/
read_minutes: 3
---

# Migrating the Hard Ones: 200-Column Teradata Tables and Geospatial
{: .no_toc }

*Estimated read: 3 min*

Most tables in a legacy estate translate the way the previous four lectures describe: run the
analyzer, run the converter, audit the diff, fix what's flagged, move on. Every migration also has
a handful of tables that don't fit that rhythm at all. Two recurring shapes are worth walking
through specifically, because both show up in nearly every Teradata- or Oracle-origin migration.

## The 200-column monolith table

Wide fact or "master record" tables -- 150, 200, sometimes 300+ columns, accumulated over a decade
of "just add a column" requests -- are common in Teradata EDWs and mechanically painful rather than
conceptually hard. The failure mode isn't that any single column is difficult to convert; it's that
manual review at that scale doesn't scale:

- **The type-mapping matrix has to be applied programmatically, not read column by column.** Export
  the source `information_schema` (or Teradata's `DBC.ColumnsV`) for the table, join it against your
  mapping matrix as a lookup table, and generate a first-pass `CREATE TABLE` from that join --
  reserve human review for the ambiguous rows (unconstrained `NUMBER`, `PERIOD` types, anything the
  join can't resolve automatically) rather than all 200.
- **Column-name collisions surface at this scale that never show up in a 15-column table.** Teradata
  and Oracle both tolerate identifiers that differ only by case or that use reserved words with
  quoting; Databricks column naming is more restrictive in a few of those edge cases, and a
  programmatic rename pass needs a deterministic, documented collision-resolution rule (append a
  suffix, not a manual judgment call per column) so the mapping is reproducible.
- **Consider whether 200 columns is a schema you actually want to carry forward**, rather than
  migrating it unchanged. A table that accreted columns for a decade in an OLTP-adjacent warehouse
  is often a candidate for splitting into a narrower core table plus one or more `STRUCT`-typed or
  vertically-partitioned extension tables during the medallion silver layer -- a re-architecture
  decision, not a translation one, and one to flag during the 3-R decision rather than silently
  deciding mid-conversion.

## Geospatial types

Oracle Spatial's `SDO_GEOMETRY`, Teradata's `ST_GEOMETRY`, and SQL Server's `GEOGRAPHY`/`GEOMETRY`
types don't map cleanly onto Delta's classic type system. Recent Databricks Runtime versions add
native `GEOMETRY`/`GEOGRAPHY` types, but a large share of migration targets still run on runtimes or
table configurations that predate them -- so the portable, engine-version-independent pattern that
works everywhere is:

```sql
CREATE TABLE store_locations (
  store_id BIGINT,
  store_name STRING,
  geom_wkt STRING,      -- Well-Known Text representation
  geom_wkb BINARY       -- Well-Known Binary, if downstream tooling needs it
) CLUSTER BY (store_id);
```

Store the geometry as **WKT (Well-Known Text)** or **WKB (Well-Known Binary)** -- both are
open, engine-agnostic geospatial serialization standards, not a Databricks-specific
workaround -- and operate on it using Databricks' built-in
[`ST_*` geospatial SQL functions](https://docs.databricks.com/aws/en/sql/language-manual/functions/st_geomfromwkt),
which parse WKT/WKB directly for distance, containment, and intersection operations without
needing a dedicated geometry column type.

The conversion itself is a data transformation, not a DDL translation -- Oracle's
`SDO_GEOMETRY.GET_WKT()` or the equivalent Teradata/SQL Server export function needs to run as part
of the extract, producing WKT strings that land in the `STRING` column above. Lakebridge's
transpiler handles the DDL shape (the column becomes `STRING`/`BINARY`); it does not rewrite the
spatial *query* logic that consumed `SDO_GEOMETRY` operators like `SDO_RELATE` -- those queries need
to be manually rewritten against `ST_*` functions, which is exactly the kind of judgment call the
80/20 lecture back in section 1 flagged as outside any transpiler's reach.

{: .important }
> Wide tables and geospatial columns share a lesson: past a certain complexity threshold, "convert
> the DDL" stops being the hard part of the migration. The hard part is deciding whether to carry
> the legacy shape forward unchanged or use the migration as the forcing function to fix a design
> that's been quietly accumulating debt for years -- and that decision belongs to the migration
> architect, not the transpiler.

<!-- prevnext:start -->

---

| [&larr; Previous: Auditing Generated DDL for Semantic Drift]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/auditing-generated-ddl-for-semantic-drift/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

