---
title: "Stored Procedures Are Not SQL"
parent: "The Procedure Autopsy: Decomposing PL/SQL"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/stored-procedures-are-not-sql/
read_minutes: 3
---

# Stored Procedures Are Not SQL
{: .no_toc }

*Estimated read: 3 min*

Schema translation and physical design both operate on tables -- static, declarative objects that
map cleanly from Oracle's `CREATE TABLE` to Delta's. A stored procedure is a different kind of
migration problem entirely, and treating it like "one more SQL object to translate" is how migration
timelines quietly blow up in the stored-procedure phase. **A stored procedure is a process with
state, not a query.** It executes statements in sequence, holds variables and cursor positions in
memory across that sequence, branches on conditions, and can leave a transaction half-committed if
something fails partway through. None of that has a direct equivalent in a single Spark SQL
statement, because Spark SQL statements don't have state between them -- each one runs, produces a
result, and forgets everything it knew.

## The four things that don't map one-to-one

- **Explicit cursors.** A PL/SQL `CURSOR` that fetches rows one at a time and processes each with
  procedural logic is a row-by-row loop -- exactly the pattern Spark's distributed, set-based engine
  is built to avoid. A migrated cursor loop that still fetches and processes one row at a time
  defeats the point of moving to a distributed engine; the Autopsy method in the next lecture exists
  to catch this before it ships as a slow PySpark `for` loop over a `collect()`.
- **Multi-statement transactions.** Oracle and SQL Server procedures routinely wrap several DML
  statements in one transaction with an explicit `COMMIT` or `ROLLBACK`. Delta Lake gives you
  transactional guarantees per table operation (a `MERGE`, an `UPDATE`), but there's no
  cross-statement transaction wrapper spanning multiple table writes the way a legacy procedure
  assumes -- multi-table atomicity has to be redesigned, not transliterated.
- **Session-scoped temp tables.** A `CREATE GLOBAL TEMPORARY TABLE` that exists only for the
  duration of a session, holding intermediate results between steps of a procedure, has no session
  concept to attach to in a Spark job -- the intermediate result needs to become a DataFrame, a
  temporary view, or a real Delta table with its own lifecycle.
- **Procedural control flow.** `IF/ELSE`, `WHILE`, `LOOP`, and `GOTO`-style branching inside a
  procedure body are imperative instructions about *when* to run which statement -- something a
  declarative SQL query has no syntax for at all. This logic has to move somewhere: into PySpark
  control flow, into a Lakeflow Declarative Pipeline's flow definitions, or into an orchestration
  layer like Lakeflow Jobs, depending on what the branching actually decides.

## Why this changes how you read the code

The autopsy work in this section isn't about producing a PySpark translation line-by-line -- it's
about first identifying *which category* each block of a procedure falls into, because a cursor loop,
a temp-table handoff, and a control-flow branch each need a different target pattern on Databricks.
Skipping straight to translation without that classification step is how procedures end up
"migrated" as literal, slow, row-at-a-time PySpark ports of code that was never meant to run that
way.

{: .important }
> Resist the urge to open a stored procedure and start translating statement by statement. Read it
> first as a process -- what state it holds, where it branches, what it commits and when -- because
> that shape, not the individual SQL statements inside it, is what determines the right Databricks
> pattern for each piece.

The next lecture turns this classification into a repeatable technique: marking up every block of a
procedure by category before writing a single line of PySpark.

<!-- prevnext:start -->

---

| [&larr; Previous: The Procedure Autopsy: Decomposing PL/SQL]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/' | relative_url }}) | [Next: The Autopsy Method: Color-Coding Every Block &rarr;]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/the-autopsy-method-color-coding-every-block/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

