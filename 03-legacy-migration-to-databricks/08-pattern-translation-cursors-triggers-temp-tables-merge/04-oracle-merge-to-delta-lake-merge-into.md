---
title: "Oracle MERGE to Delta Lake MERGE INTO"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/oracle-merge-to-delta-lake-merge-into/
read_minutes: 3
---

# Oracle MERGE to Delta Lake MERGE INTO
{: .no_toc }

*Estimated read: 3 min*

Of every pattern in this section, `MERGE` is the one that needs the least conceptual translation --
Oracle's `MERGE` and Delta's **[`MERGE
INTO`](https://docs.databricks.com/aws/en/delta/merge)** solve the same upsert problem with nearly
identical syntax. The differences that matter are smaller and easier to miss precisely because the
statements look so similar.

## The direct mapping

```sql
-- Oracle
MERGE INTO orders tgt
USING stg_orders src
ON (tgt.order_id = src.order_id)
WHEN MATCHED THEN
  UPDATE SET tgt.status = src.status, tgt.updated_at = SYSDATE
WHEN NOT MATCHED THEN
  INSERT (order_id, status, updated_at) VALUES (src.order_id, src.status, SYSDATE);
```

```sql
-- Delta Lake
MERGE INTO orders AS tgt
USING stg_orders AS src
ON tgt.order_id = src.order_id
WHEN MATCHED THEN
  UPDATE SET tgt.status = src.status, tgt.updated_at = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (order_id, status, updated_at) VALUES (src.order_id, src.status, current_timestamp());
```

Target, source, join condition, matched and not-matched branches -- the structure carries over almost
verbatim. `SYSDATE` becomes `current_timestamp()`, and that's close to the entire syntactic delta
for the common case.

## Where Delta's `MERGE INTO` goes further

- **`WHEN NOT MATCHED BY SOURCE`.** Oracle's `MERGE` has no clause for "target rows with no matching
  source row." Delta adds one, which turns a Oracle pattern that often required a separate `DELETE`
  statement (or a full-outer-join workaround) into one statement:

  ```sql
  MERGE INTO orders AS tgt
  USING stg_orders AS src
  ON tgt.order_id = src.order_id
  WHEN MATCHED THEN UPDATE SET tgt.status = src.status
  WHEN NOT MATCHED THEN INSERT (order_id, status) VALUES (src.order_id, src.status)
  WHEN NOT MATCHED BY SOURCE THEN UPDATE SET tgt.status = 'CANCELLED';
  ```

- **Conditional clauses.** Both platforms support `WHEN MATCHED AND <condition>`, but Delta allows
  multiple `WHEN MATCHED` clauses evaluated in order, letting one `MERGE INTO` statement replace
  several sequential Oracle statements that handled different match sub-cases separately.
- **Schema evolution.** `MERGE INTO ... WHEN NOT MATCHED THEN INSERT *` combined with
  `spark.databricks.delta.schema.autoMerge.enabled = true` lets new source columns flow into the
  target table's schema automatically -- there's no Oracle equivalent, since Oracle's `MERGE` target
  schema is always fixed in advance.

## The idempotency check that's easy to skip

A `MERGE INTO` re-run against the same source batch after a partial failure should produce the same
result as running it once -- but only if the join condition uniquely identifies each target row. A
join condition with duplicate matches on the source side causes Delta to throw an error rather than
silently apply nondeterministic updates, which is a stricter (and safer) behavior than some legacy
engines enforce by default.

{: .important }
> Before migrating any `MERGE` statement literally, check the source side for duplicate keys against
> the join condition. Delta's `MERGE INTO` will reject an ambiguous match rather than silently
> picking one row, which surfaces a data-quality issue the legacy `MERGE` may have been quietly
> masking for years.

`MERGE INTO` is also the mechanism the next lecture builds on directly -- **SCD Type 2** history
tracking, one of the most common patterns built on top of an upsert in any legacy warehouse.

<!-- prevnext:start -->

---

| [&larr; Previous: Temp Table to CTE or Lakeflow Declarative Pipeline]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/temp-table-to-cte-or-lakeflow-declarative-pipeline/' | relative_url }}) | [Next: SCD Type 2 the Lakehouse Way &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/scd-type-2-the-lakehouse-way/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

