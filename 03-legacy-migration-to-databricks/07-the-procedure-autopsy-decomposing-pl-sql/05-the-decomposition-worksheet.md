---
title: "The Decomposition Worksheet"
parent: "The Procedure Autopsy: Decomposing PL/SQL"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/the-decomposition-worksheet/
read_minutes: 3
---

# The Decomposition Worksheet
{: .no_toc }

*Estimated read: 3 min*

This section has built up three separate analytical steps -- color-coding a procedure's blocks,
separating business logic from procedural scaffolding, and graphing a trigger cascade's true
execution order -- without a single artifact to carry the results of all three forward. The
**decomposition worksheet** is that artifact: one page per procedure or trigger package, filled out
during the autopsy, that becomes the handoff document into the Pattern Translation section that
follows this one.

## The worksheet

| Column | Captures | Filled in during |
|---|---|---|
| **Block ID & color** | Line range and category (blue/green/yellow/orange/red) | Autopsy method pass |
| **Business logic (Y/N)** | Whether this block encodes a rule the business depends on, or is procedural mechanism | Business-logic classification pass |
| **Extracted rule** | The rule itself, stated as a plain sentence, for blocks marked Y | Business-logic classification pass |
| **Upstream / downstream dependency** | Which tables this block reads from and writes to, and which trigger (if any) fires next | Trigger-graph mapping pass, where applicable |
| **Target pattern** | `MERGE INTO`, Lakeflow Declarative Pipeline flow, PySpark control flow, or Lakeflow Jobs orchestration | Filled in last, once the first four columns are complete |

The order of the columns matters: **target pattern is always the last column filled in**, not the
first. A common failure mode when procedure migration is under deadline pressure is picking the
target pattern before finishing the classification -- deciding "this becomes a `MERGE`" while still
reading the procedure, then discovering three blocks later that a cursor loop's real logic doesn't
fit that shape after all, and reworking already-written code. The worksheet's column order forces the
classification to finish before any translation commitment is made.

## Worked example row

For the small procedure used across this section's earlier lectures:

| Block ID & color | Business logic | Extracted rule | Dependency | Target pattern |
|---|---|---|---|---|
| L1-L3, green + red | N | -- (mechanism only) | reads `stg_orders` | discarded -- absorbed into target pattern below |
| L2, blue (inner statement) | Y | "An order in `stg_orders` with a non-null `order_id` becomes `PROCESSED`" | writes `orders` | `MERGE INTO orders ... WHEN MATCHED THEN UPDATE SET status = 'PROCESSED'` |
| L4, yellow | N | -- (batch-commit mechanism, unnecessary on Delta) | -- | none required |

One filled-out row like this, per procedure, is what a migration team hands to whoever writes the
actual PySpark or Lakeflow Declarative Pipeline code -- and it's also what makes a code reviewer's
job tractable: they can check the target pattern against the extracted rule without re-reading the
original PL/SQL at all.

{: .important }
> Fill out the worksheet **before** any translation code is written, the same way the physical
> design decision card from the previous section gets filled out before a table's `CREATE TABLE`
> statement. A worksheet completed after the fact, to document a translation someone already wrote
> from memory, catches far fewer of the mismatches this exercise exists to prevent.

With the worksheet complete for every procedure and trigger package in the inventory, the next
section turns the "target pattern" column into working code -- the specific translation patterns for
cursors, triggers, temp tables, and `MERGE` logic this section's examples have been pointing toward.

<!-- prevnext:start -->

---

| [&larr; Previous: Mapping a Cascading-Trigger Package]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/mapping-a-cascading-trigger-package/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

