---
title: "Auditing Generated DDL for Semantic Drift"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/auditing-generated-ddl-for-semantic-drift/
read_minutes: 3
---

# Auditing Generated DDL for Semantic Drift
{: .no_toc }

*Estimated read: 3 min*

Precision loss is a numeric-column problem. **Semantic drift** is the broader version of the same
risk across an entire `CREATE TABLE` statement: the converted DDL parses, runs, and *looks* right,
but no longer enforces the same rules the source schema did. It's the difference between "this
table has the same columns" and "this table behaves the same way when bad data shows up."

## What gets lost silently in translation

Run a systematic diff between source and converted DDL, checking specifically for these, because
none of them cause a syntax error when dropped:

- **`NOT NULL` constraints.** Straightforward to translate and almost always preserved correctly --
  but worth a first pass anyway, since a dropped `NOT NULL` on a join key won't be caught by
  anything except a downstream null-related join failure weeks later.
- **`CHECK` constraints.** Oracle and SQL Server `CHECK` constraints that reference built-in
  functions (`REGEXP_LIKE`, date arithmetic, engine-specific string functions) frequently have no
  direct Databricks SQL equivalent. The converter's safest move is to comment them out and flag
  them for manual review -- which means the validation logic that used to live *in the database*
  is now missing, silently, unless someone reads the flag.
- **`DEFAULT` expressions tied to sequences or engine functions.** `DEFAULT nextval('seq_orders')`
  or `DEFAULT SYSDATE` have no drop-in equivalent; Databricks handles surrogate keys and generated
  timestamps differently (identity columns, or generation logic in the pipeline itself). A
  converted table missing its default doesn't error -- it just starts accepting `NULL` where a
  generated value used to appear automatically.
- **Foreign keys and referential integrity.** Delta Lake does not enforce foreign keys at write
  time the way Oracle or SQL Server do. A converted schema that drops FK constraints hasn't broken
  anything mechanically -- but the safety net that used to *reject* an orphaned row at insert time
  is gone, and nothing tells you that unless you go looking.
- **Column-level collation and case-sensitivity rules.** A SQL Server column with a
  case-insensitive collation feeding a `WHERE code = 'ABC'` filter behaves differently once it's a
  Databricks `STRING`, which compares case-sensitively by default -- a query that used to match
  `'abc'` and `'ABC'` alike now only matches one.

## A practical audit pattern

Don't audit by reading the converted DDL in isolation -- diff it against the source, object by
object, with the constraint categories above as your checklist:

```bash
# Pseudocode for the review loop, not a single tool -- Lakebridge produces the
# converted DDL; the source-vs-target diff is a manual or lightly-scripted step.
for table in converted/databricks_sql/*.sql; do
  diff <(extract_constraints "$table") <(extract_constraints "source_ddl/$(basename "$table")")
done
```

Track findings the same way you'd track any code review: a checklist per table of "constraint
dropped / constraint requires manual reimplementation / constraint verified equivalent," not a
one-time pass that gets thrown away once the table loads successfully.

{: .important }
> A converted table that loads data without error has passed a much lower bar than "matches the
> source schema's behavior." Treat every dropped constraint as a design decision that needs an
> explicit owner and a replacement plan -- reimplemented as a Delta `CHECK` constraint, enforced in
> the pipeline, or consciously accepted as no longer needed -- not a gap that resolves itself.

<!-- prevnext:start -->

---

| [&larr; Previous: The Silent Precision-Loss Bug]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/the-silent-precision-loss-bug/' | relative_url }}) | [Next: Migrating the Hard Ones: 200-Column Teradata Tables and Geospatial &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/migrating-the-hard-ones-200-column-teradata-tables-and-geospatial/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

