---
title: "Check Your Knowledge"
parent: "The Procedure Autopsy: Decomposing PL/SQL"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of this section's procedure-autopsy concepts before moving on.

1. Why is a stored procedure a fundamentally different migration problem than a table?
   - A. Procedures are always shorter than table definitions
   - B. A procedure is a stateful process that executes in sequence and can branch, not a single declarative statement
   - C. Procedures never contain any SQL
   - D. Tables require more testing than procedures do

2. Which of the following does NOT have a direct one-to-one equivalent in Spark SQL?
   - A. A `SELECT` statement with a `WHERE` clause
   - B. An explicit cursor that fetches and processes rows one at a time
   - C. A `JOIN` between two tables
   - D. A `GROUP BY` aggregation

3. In the Autopsy method's color scheme, what does a yellow block represent?
   - A. Pure set-based SQL
   - B. A transaction boundary (`COMMIT`/`ROLLBACK`/`SAVEPOINT`)
   - C. A session-scoped temp table
   - D. Procedural control flow like `IF/ELSE`

4. What does a color-coded pass reveal that a plain read of the procedure doesn't?
   - A. The exact runtime of the procedure in production
   - B. How much of the procedure is genuinely set-based (blue) versus procedural scaffolding
   - C. The original author's name
   - D. Whether the procedure has ever thrown an error

5. According to "Only the Business Logic Migrates," which block color is described as almost always business logic?
   - A. Yellow
   - B. Orange
   - C. Blue
   - D. Green

6. Why are green (cursor loop) blocks usually discarded rather than translated literally?
   - A. Because cursors are illegal in Delta Lake
   - B. Because the loop construct itself encodes no business rule -- only the statement inside it does
   - C. Because Spark cannot execute any loop-like logic
   - D. Because cursors always contain syntax errors

7. What makes a cascading-trigger package harder to autopsy than a single stored procedure?
   - A. Triggers cannot contain any SQL statements
   - B. The execution order is implicit, scattered across separate trigger definitions, and has to be reconstructed as a graph
   - C. Triggers are always written in a different language than procedures
   - D. Databricks does not allow reading trigger source code

8. In the trigger-graph mapping approach, what does a topological sort of the graph produce?
   - A. A list of every column referenced in any trigger
   - B. The true execution order the cascading triggers implied
   - C. A count of how many triggers exist in the schema
   - D. The original CREATE TRIGGER timestamps

9. On the decomposition worksheet, why is "target pattern" always the last column filled in?
   - A. It's the least important column
   - B. Committing to a target pattern before classification is finished risks reworking code once a later block doesn't fit the chosen shape
   - C. Target patterns can only be assigned by a database administrator
   - D. The worksheet template requires columns to be filled alphabetically

10. What is the decomposition worksheet's role in the migration process?
    - A. A one-time compliance form with no further use
    - B. A per-procedure handoff document connecting the autopsy's classification work to the actual translation work in the next section
    - C. A replacement for reading the original PL/SQL entirely
    - D. A performance benchmarking report

## Answer Key

1. **B** -- A procedure executes statements in sequence, holds state, and branches -- properties a single declarative SQL statement doesn't have.
2. **B** -- Explicit row-at-a-time cursors have no direct equivalent in Spark's set-based, distributed execution model.
3. **B** -- Yellow marks transaction boundaries like `COMMIT`, `ROLLBACK`, and `SAVEPOINT`.
4. **B** -- The color pass produces a visual map of how much of the procedure is genuinely set-based SQL versus procedural mechanism.
5. **C** -- Blue blocks (pure set-based SQL) are almost always the actual business rules worth preserving.
6. **B** -- The loop mechanism itself is procedural plumbing; only the condition and action inside the loop body carry business meaning.
7. **B** -- Trigger execution order is an emergent property of separate `CREATE TRIGGER` definitions, discoverable only by reconstructing the dependency graph.
8. **B** -- A topological sort of the trigger dependency graph produces the true execution order the cascade implied.
9. **B** -- Filling in target pattern last ensures classification is complete before any translation commitment is made, avoiding rework.
10. **B** -- The worksheet is the per-procedure artifact that carries the autopsy's findings forward into the Pattern Translation section's actual code.

<!-- prevnext:start -->

---

| [&larr; Previous: The Decomposition Worksheet]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/the-decomposition-worksheet/' | relative_url }}) | [Next: Pattern Translation: Cursors, Triggers, Temp Tables, MERGE &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

