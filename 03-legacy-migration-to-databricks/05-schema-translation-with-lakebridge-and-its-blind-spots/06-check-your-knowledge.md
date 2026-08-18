---
title: "Check Your Knowledge"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you've learned from this section before moving on to physical design for Delta.

1. Why is Oracle's `DATE` type a common source of silent migration bugs when mapped to Delta?
   A. Oracle `DATE` cannot store negative years
   B. Oracle `DATE` always includes a time component, but Delta `DATE` does not, so mapping it to `DATE` silently truncates the time
   C. Oracle `DATE` is deprecated and should never be migrated
   D. Delta has no `DATE` type at all

2. What is the safer default mapping for an unconstrained Oracle `NUMBER` column, and why?
   A. `DOUBLE`, because it requires no precision or scale
   B. `STRING`, to avoid any type coercion
   C. A fixed-precision `DECIMAL`, because `DOUBLE`'s binary floating point representation risks rounding error on aggregates
   D. `BOOLEAN`, since most `NUMBER` columns are flags

3. In the Lakebridge workflow, what does the **analyzer** produce that the **converter** does not?
   A. The final converted Databricks SQL DDL
   B. An object inventory, complexity/effort scoring, and cross-object dependencies -- run before conversion, not as a conversion step
   C. A live connection to the source database
   D. A reconciliation report comparing row counts

4. Which three transpiler engines does Lakebridge's converter select between?
   A. Spark, Hive, and Presto
   B. BladeBridge, Morpheus, and Switch
   C. Sqoop, Flume, and NiFi
   D. Oracle GoldenGate, Qlik Replicate, and Fivetran

5. Why does an undersized target `DECIMAL(p,s)` cause a worse problem than an oversized one?
   A. It causes the load job to run out of memory
   B. It silently rejects or rounds values that exceed its precision, and a rounded financial value doesn't raise an alarm the way a rejected row does
   C. It prevents the table from being created at all
   D. It has no effect on numeric accuracy

6. According to the DDL-auditing lecture, why does Delta Lake not catching dropped foreign keys as an error still matter?
   A. Foreign keys are automatically reimplemented by the converter
   B. Delta doesn't enforce referential integrity at write time, so a dropped FK removes a safety net without any error signaling that loss
   C. Foreign keys are irrelevant to any analytical workload
   D. Databricks rejects any schema that lacks foreign keys

7. What commonly happens to source `CHECK` constraints that reference engine-specific functions during conversion?
   A. They are automatically rewritten to equivalent Databricks SQL with no review needed
   B. They are silently dropped or commented out by the converter, and validation logic that lived in the database is now missing unless flagged
   C. They cause the entire conversion job to fail
   D. They are converted into Delta `NOT NULL` constraints

8. For a 200-column legacy table, what does this section recommend over reviewing all 200 columns manually?
   A. Migrating the table as a single `STRING` blob column
   B. Skipping type mapping entirely for wide tables
   C. Applying the type-mapping matrix programmatically via a metadata join, reserving manual review for ambiguous rows only
   D. Splitting the table into 200 separate single-column tables

9. What is the standard, engine-version-independent pattern for migrating Oracle `SDO_GEOMETRY` or Teradata `ST_GEOMETRY` columns?
   A. Drop the geometry column entirely since Delta cannot represent spatial data
   B. Store the geometry as WKT or WKB in a `STRING`/`BINARY` column and operate on it with Databricks `ST_*` functions
   C. Convert geometry to a `DOUBLE` representing distance from the equator
   D. Store the geometry as a Python pickle in a `BINARY` column

10. Why does the section describe Lakebridge's transpiler as handling the DDL shape but not the spatial query logic for geospatial columns?
    A. Because spatial queries never need to change during a migration
    B. Because functions like Oracle's `SDO_RELATE` have no automatic transpilation path and must be manually rewritten against `ST_*` functions
    C. Because Lakebridge does not support geospatial source columns at all
    D. Because Databricks does not support any spatial functions

## Answer Key

1. **B** -- Oracle `DATE` stores a time component that Delta `DATE` cannot hold, so the mapping silently truncates it; mapping to `TIMESTAMP` avoids the loss.
2. **C** -- A fixed-precision `DECIMAL` avoids the binary floating-point rounding error that `DOUBLE` introduces on financial aggregates.
3. **B** -- The analyzer inventories objects and estimates complexity/effort/dependencies as a pre-conversion assessment step, separate from the converter's DDL translation.
4. **B** -- BladeBridge, Morpheus, and Switch are Lakebridge's three transpiler engines, selected via the transpiler configuration.
5. **B** -- Undersized `DECIMAL` precision causes silent rounding or rejection, and a quietly rounded financial figure is more dangerous than a rejected row because nothing flags it.
6. **B** -- Delta's lack of write-time FK enforcement means a dropped constraint removes protection against orphaned rows without any error surfacing that fact.
7. **B** -- Engine-specific `CHECK` constraints are typically commented out or dropped, removing validation logic silently unless the audit process catches it.
8. **C** -- Programmatically joining the source metadata catalog against the type-mapping matrix scales to wide tables; manual review is reserved for genuinely ambiguous columns.
9. **B** -- WKT/WKB stored in `STRING`/`BINARY` plus Databricks `ST_*` functions is the portable pattern that works regardless of native geometry type support.
10. **B** -- Spatial operators like `SDO_RELATE` have no mechanical equivalent; the underlying query logic must be manually rewritten, not just the column's DDL shape.

<!-- prevnext:start -->

---

| [&larr; Previous: Migrating the Hard Ones: 200-Column Teradata Tables and Geospatial]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/migrating-the-hard-ones-200-column-teradata-tables-and-geospatial/' | relative_url }}) | [Next: Physical Design for Delta: Liquid Clustering Over Indexes &rarr;]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

