---
title: "SCD Type 2 the Lakehouse Way"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/scd-type-2-the-lakehouse-way/
read_minutes: 3
---

# SCD Type 2 the Lakehouse Way
{: .no_toc }

*Estimated read: 3 min*

**SCD Type 2** -- keeping a full history of a dimension row's changes, each version stamped with its
own effective and expiry dates -- is one of the most common patterns built directly on top of
`MERGE`, and it's usually the messiest procedure in a legacy schema: a cursor that finds changed
rows, a separate `UPDATE` to close out the old version, and a separate `INSERT` for the new one, all
wrapped in a transaction to keep the two writes consistent. Delta Lake collapses that into either one
`MERGE INTO` statement or a single declarative flow.

## SCD Type 2 as one MERGE

The trick is that a single `MERGE INTO` can carry **two different `WHEN MATCHED` clauses**, evaluated
in order -- one to close the old version, one to catch true no-ops:

```sql
MERGE INTO customer_dim AS tgt
USING customer_updates AS src
ON tgt.customer_id = src.customer_id AND tgt.is_current = true
WHEN MATCHED AND tgt.email != src.email THEN
  UPDATE SET tgt.is_current = false, tgt.end_date = current_date()
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, is_current, start_date, end_date)
  VALUES (src.customer_id, src.email, true, current_date(), NULL);
```

This closes the old row (`is_current = false`) when a tracked column actually changed, and inserts
the new current row -- but notice it only inserts for genuinely new customers, not for existing
customers whose current row just got closed. A closed row needs its replacement inserted too, which
is why most real implementations run this as two passes over the same source, or reach for the
higher-level abstraction below instead of hand-rolling the two-pass logic themselves.

## AUTO CDC: the pattern built into the platform

Because SCD Type 2 is common enough to standardize, Lakeflow Declarative Pipelines expose it
directly through the **[AUTO CDC
API](https://docs.databricks.com/aws/en/dlt/cdc)** (`create_auto_cdc_flow`, the successor to the
older `APPLY CHANGES INTO` syntax), which handles the close-and-insert pairing, out-of-order change
records, and duplicate detection without hand-written `MERGE` logic at all:

```python
import dlt

dlt.create_streaming_table("customer_dim")

dlt.create_auto_cdc_flow(
    target="customer_dim",
    source="customer_updates",
    keys=["customer_id"],
    sequence_by="updated_at",
    stored_as_scd_type=2,
)
```

Three lines of configuration -- keys, ordering column, and `stored_as_scd_type=2` -- replace what a
legacy procedure implemented as a cursor, two DML statements, and a transaction wrapper, and the
pipeline handles late-arriving or out-of-order updates correctly by construction, using
`sequence_by` rather than assuming updates always arrive in order the way a row-level trigger did.

{: .important }
> Reach for `create_auto_cdc_flow` over a hand-written two-pass `MERGE` whenever the source can
> genuinely arrive out of order -- CDC feeds, multi-region writes, replayed batches. A hand-rolled
> `MERGE` that assumes strictly ordered arrival will silently produce a wrong history the first time
> that assumption breaks; the AUTO CDC API is built specifically to handle it correctly.

With cursors, triggers, temp tables, `MERGE`, and SCD Type 2 all covered, the next lecture catalogs
the mistakes migration teams make translating these patterns literally instead of idiomatically --
worth reading even for patterns already migrated, as a check against what shipped.

<!-- prevnext:start -->

---

| [&larr; Previous: Oracle MERGE to Delta Lake MERGE INTO]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/oracle-merge-to-delta-lake-merge-into/' | relative_url }}) | [Next: Anti-Pattern Gallery and Pattern Cheat Sheet &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/anti-pattern-gallery-and-pattern-cheat-sheet/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

