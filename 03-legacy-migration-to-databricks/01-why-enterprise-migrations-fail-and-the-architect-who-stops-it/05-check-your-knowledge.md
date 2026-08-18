---
title: "Check Your Knowledge"
parent: "Why Enterprise Migrations Fail (And the Architect Who Stops It)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you've learned from this section before moving on to profiling the legacy monolith.

1. According to this section, what is the most common *root cause* behind stalled enterprise EDW migrations?
   A. Choosing the wrong cloud region
   B. Treating the project as pure data movement while underestimating the procedural/business-logic layer
   C. Databricks compute costs being higher than expected
   D. Insufficient network bandwidth between source and target

2. Which of the following is explicitly named as one of the three most common causes of migration failure?
   A. Reconciliation being treated as an afterthought instead of being built in from day one
   B. Choosing Delta Lake over Parquet
   C. Running too many parallel Databricks clusters
   D. Migrating Unity Catalog before schema translation

3. Lakebridge's three main phases are assessment, transpilation, and reconciliation. Which best describes the transpilation phase?
   A. Estimating the total cost of ownership of the legacy platform
   B. Converting SQL and orchestration logic from source dialects into Databricks-compatible ANSI SQL
   C. Validating that migrated data matches the source system
   D. Migrating Oracle role grants into Unity Catalog

4. What does the Lakebridge Agentic Converter add beyond core Lakebridge transpilation?
   A. A billing dashboard for chargeback reporting
   B. Project-based conversion with iterative validation/retry and reusable custom "skills" for business rules
   C. A replacement for Unity Catalog governance
   D. Automatic stored procedure decomposition with no human review needed

5. Why can't a SQL transpiler alone fully migrate a stored procedure that opens a cursor and conditionally calls other procedures based on runtime state?
   A. Transpilers cannot parse PL/SQL syntax at all
   B. That kind of procedural, stateful logic requires decomposition into set-based operations, which is a design decision rather than a syntax conversion
   C. Cursors are deprecated in Delta Lake
   D. Databricks does not support conditional logic in PySpark

6. In the migration architect's six-stage operating model, which stage comes immediately before "Ingest"?
   A. Govern and optimize
   B. Assess
   C. Translate
   D. Reconcile and cut over

7. Per the operating model lecture, why must a workload be reconciled before it is cut over, even under schedule pressure?
   A. Because Unity Catalog requires reconciliation logs to grant permissions
   B. Because skipping reconciliation is one of the most common causes of trust breakdown and stalled migrations described earlier in the section
   C. Because Lakebridge will not run without a completed reconciliation job
   D. Because reconciliation is a legal requirement for all cloud migrations

8. Which artifact in the downloadable toolkit is specifically built to replace a legacy role hierarchy of hundreds of database roles?
   A. The TCO calculator
   B. The go/no-go decision matrix
   C. The ABAC tag taxonomy
   D. The reconciliation script library

9. What is the purpose of the go/no-go decision matrix introduced in this section?
   A. To estimate cloud compute costs before migration begins
   B. To turn a cutover decision into a structured, defensible checklist instead of a gut call under pressure
   C. To automatically translate PL/SQL into PySpark
   D. To assign Unity Catalog privileges to service principals

10. Why does this section frame Lakebridge as "a very capable junior engineer" rather than a full replacement for a migration architect?
    A. Because Lakebridge is not yet publicly available
    B. Because it produces fast first-pass translations but cannot verify semantic correctness or business intent on its own
    C. Because Lakebridge only works with Snowflake as a source
    D. Because Lakebridge requires a paid enterprise license to run

## Answer Key

1. **B** -- The section frames the graveyard as a business-logic archaeology problem, not a data-movement problem; procedural logic is consistently underestimated.
2. **A** -- Reconciliation-as-afterthought is one of the three named causes, alongside an unprofiled estate and blanket lift-and-shift.
3. **B** -- Transpilation converts source-dialect SQL/orchestration into Databricks-compatible ANSI SQL; assessment estimates effort/TCO and reconciliation validates data.
4. **B** -- The Agentic Converter organizes work into projects with iterative validate/retry and supports custom "skills" for encoding business rules.
5. **B** -- Stateful, side-effecting procedural logic needs to be redesigned as set-based DataFrame operations, which a mechanical transpiler can't infer.
6. **C** -- The order is Assess, Decide, Translate, Ingest, Reconcile and cut over, Govern and optimize -- Translate precedes Ingest.
7. **B** -- The operating-model lecture ties skipping reconciliation directly back to the graveyard causes named in the first lecture.
8. **C** -- The ABAC tag taxonomy replaces a role-explosion hierarchy (e.g., five hundred Oracle roles) with governed tags in Unity Catalog.
9. **B** -- The matrix turns cutover into an auditable, structured decision covering reconciliation status, rollback readiness, sign-off, and compute headroom.
10. **B** -- Lakebridge produces strong first-pass output quickly but cannot sign off on semantic correctness or infer undocumented business intent; that remains the architect's job.

<!-- prevnext:start -->

---

| [&larr; Previous: How This Course Works and Your Downloadable Toolkit]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/how-this-course-works-and-your-downloadable-toolkit/' | relative_url }}) | [Next: The Autopsy: Profiling the Legacy Monolith &rarr;]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

