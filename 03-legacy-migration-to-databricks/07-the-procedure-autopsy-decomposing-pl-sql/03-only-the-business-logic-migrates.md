---
title: "Only the Business Logic Migrates"
parent: "The Procedure Autopsy: Decomposing PL/SQL"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/only-the-business-logic-migrates/
read_minutes: 3
---

# Only the Business Logic Migrates
{: .no_toc }

*Estimated read: 3 min*

A color-coded procedure makes a fact visible that plain reading usually hides: most of a legacy
procedure's line count isn't business logic at all. It's scaffolding the procedural language forced
on the author -- cursor bookkeeping to fake set-based processing, `COMMIT`/`ROLLBACK` calls to
approximate atomicity the platform didn't provide natively, temp-table staging to work around the
absence of intermediate DataFrames. **Only the blue blocks -- the actual business rules -- are worth
migrating. Everything else is a workaround for constraints Databricks doesn't have.**

## Reading a procedure for what to keep

Take the color-coded pass from the previous lecture and ask one question of each block: *does this
block encode a decision the business cares about, or does it exist to manage the mechanics of the
procedural language?*

- **Blue blocks are almost always business logic.** `UPDATE orders SET status = 'PROCESSED' WHERE
  ...` encodes a real rule about what makes an order processed. That rule has to survive the
  migration in some form, even if its surrounding mechanics change completely.
- **Yellow blocks (transaction boundaries) are rarely business logic.** A `COMMIT` after every
  thousand rows in a legacy procedure is usually about managing rollback-segment or log-file growth
  on the source engine -- a constraint that doesn't exist the same way in Delta Lake's
  transaction model. Carrying that batching logic forward literally, instead of relying on
  `MERGE INTO`'s native atomicity, adds complexity for no benefit.
- **Green blocks (cursor loops) are a mechanism, not a rule.** The row-at-a-time `FOR r IN (SELECT
  ...) LOOP` construct itself encodes nothing about the business -- only the condition being checked
  and the action being taken inside the loop body do. The loop can usually be discarded entirely once
  its inner blue statement is rewritten as a set-based `MERGE` or `UPDATE`.
- **Orange blocks (temp tables) are usually plumbing.** A session-scoped temp table that holds
  intermediate results between two steps of a procedure is there because the source language has no
  first-class notion of an in-memory DataFrame -- the equivalent on Databricks is a PySpark
  DataFrame or a temporary view, not a literal temp-table translation.

## A worked example

Recall the small example from the previous lecture:

```text
FOR r IN (SELECT order_id FROM stg_orders) LOOP
  IF r.order_id IS NOT NULL THEN
    UPDATE orders SET status = 'PROCESSED' WHERE order_id = r.order_id;
  END IF;
END LOOP;
COMMIT;
```

The green loop, the red `IF`, and the yellow `COMMIT` are all mechanism. The one piece of actual
business logic is "an order in `stg_orders` becomes `PROCESSED`" -- which survives the migration as:

```sql
UPDATE orders SET status = 'PROCESSED'
WHERE order_id IN (SELECT order_id FROM stg_orders WHERE order_id IS NOT NULL);
```

Four lines of procedural scaffolding collapsed into a single set-based statement, with the business
rule fully preserved and nothing procedural left to maintain.

{: .important }
> When a stakeholder asks "did we migrate the logic correctly," they mean the blue blocks -- the
> rules. They don't mean the cursor loop or the commit cadence. Confusing "faithfully porting every
> line" with "faithfully preserving the business rule" is what turns a two-week procedure migration
> into a two-month one, translating scaffolding that never needed to exist on the new platform.

The next lecture applies this same discipline to something structurally harder: a package of
cascading triggers, where the business logic isn't in one procedure body but scattered across a
chain of triggers that fire on each other.

<!-- prevnext:start -->

---

| [&larr; Previous: The Autopsy Method: Color-Coding Every Block]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/the-autopsy-method-color-coding-every-block/' | relative_url }}) | [Next: Mapping a Cascading-Trigger Package &rarr;]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/mapping-a-cascading-trigger-package/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

