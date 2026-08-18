---
title: "The Autopsy Method: Color-Coding Every Block"
parent: "The Procedure Autopsy: Decomposing PL/SQL"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/the-autopsy-method-color-coding-every-block/
read_minutes: 3
---

# The Autopsy Method: Color-Coding Every Block
{: .no_toc }

*Estimated read: 3 min*

The previous lecture identified four categories of logic hiding inside a typical stored procedure --
cursors, transactions, temp tables, and control flow -- alongside the pure data-transformation
statements that make up the rest. The **Autopsy method** turns that classification into a literal,
mechanical exercise: print the procedure body and mark every block with a color before writing any
migration code at all.

## The five categories

| Color | Block type | Example |
|---|---|---|
| **Blue** | Pure set-based SQL (`SELECT`, `INSERT ... SELECT`, `UPDATE ... WHERE`) | A single statement that transforms or filters rows |
| **Green** | Explicit cursor loops | `CURSOR c IS SELECT ...` followed by `FETCH`/row-at-a-time processing |
| **Yellow** | Transaction boundaries | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |
| **Orange** | Session temp tables | `CREATE GLOBAL TEMPORARY TABLE`, intermediate result staging |
| **Red** | Procedural control flow | `IF/ELSE`, `WHILE`, `LOOP`, `GOTO`, exception handlers |

Working through a real procedure line by line and assigning each block one of these five colors
takes minutes for a hundred-line procedure, and it produces something a pure code-read doesn't: a
visual map of how much of the procedure is genuinely blue. A procedure that's 80% blue with a couple
of yellow commit boundaries is a straightforward `MERGE INTO` or Lakeflow Declarative Pipeline flow.
A procedure that's mostly green and red -- cursor loops wrapped in nested conditionals -- is signaling
that its logic needs to be re-architected, not transliterated, before it belongs on Databricks at
all.

## Why color, not a checklist

A checklist tells you *that* a procedure has a cursor somewhere in it. A color-coded pass tells you
*where*, how much of the procedure it touches, and what surrounds it -- whether that cursor loop sits
inside a transaction boundary that also needs redesigning, or stands alone and can be replaced with a
single set-based statement in isolation. That structural picture is what the next lecture's "only the
business logic migrates" principle depends on: you can't tell what's business logic worth preserving
versus procedural plumbing worth discarding until you can see the whole procedure's shape at once.

```text
CREATE PROCEDURE apply_daily_orders AS
BEGIN
  -- BLUE: pure set-based staging
  INSERT INTO stg_orders SELECT * FROM raw_orders WHERE load_date = TRUNC(SYSDATE);

  -- GREEN: row-at-a-time cursor loop
  FOR r IN (SELECT order_id FROM stg_orders) LOOP
    -- RED: procedural branching inside the loop
    IF r.order_id IS NOT NULL THEN
      UPDATE orders SET status = 'PROCESSED' WHERE order_id = r.order_id;
    END IF;
  END LOOP;

  -- YELLOW: transaction boundary
  COMMIT;
END;
```

In this small example, the color pass immediately surfaces the real finding: the green/red
cursor-and-branch block is doing exactly what the blue `INSERT` block already could with a single
`UPDATE ... WHERE order_id IN (SELECT order_id FROM stg_orders)` -- no row-at-a-time loop required.
That's the kind of finding the autopsy is designed to surface before any PySpark gets written.

{: .important }
> Do the color pass on paper or in a text markup, not in your head. The value of the Autopsy method
> is in forcing yourself to categorize *every* block, including the ones that look obviously simple --
> the procedures that hide the worst migration surprises are usually the ones nobody looked at closely
> because they seemed short.

Mapping cursors and control flow to Spark-native patterns is only half the exercise -- the next
lecture covers the other half: deciding which parts of a color-coded procedure are business logic
worth carrying forward, and which are procedural scaffolding the platform makes unnecessary.

<!-- prevnext:start -->

---

| [&larr; Previous: Stored Procedures Are Not SQL]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/stored-procedures-are-not-sql/' | relative_url }}) | [Next: Only the Business Logic Migrates &rarr;]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/only-the-business-logic-migrates/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

