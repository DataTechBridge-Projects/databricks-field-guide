---
title: "Legacy Migration to Databricks"
nav_order: 5
has_children: true
permalink: /03-legacy-migration-to-databricks/
---

# Legacy Migration to Databricks

A field guide for migrating a legacy enterprise data warehouse (Oracle, Teradata, SQL Server, PL/SQL) to the Databricks Lakehouse: assessment, TCO, schema and procedure translation, reconciliation, cutover, governance migration, and FinOps.

<!-- PART-OVERVIEW: placeholder, filled in during content generation -->

## Sections

| # | Section | Items |
|---|---------|-------|
| 1 | [Why Enterprise Migrations Fail (And the Architect Who Stops It)]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/' | relative_url }}) | 5 |
| 2 | [The Autopsy: Profiling the Legacy Monolith]({{ '/03-legacy-migration-to-databricks/02-the-autopsy-profiling-the-legacy-monolith/' | relative_url }}) | 6 |
| 3 | [The 3-R Decision and the TCO That Convinces the CFO]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/' | relative_url }}) | 6 |
| 4 | [Lakehouse Federation: Migrate Without Migrating]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/' | relative_url }}) | 5 |
| 5 | [Schema Translation with Lakebridge (and Its Blind Spots)]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/' | relative_url }}) | 6 |
| 6 | [Physical Design for Delta: Liquid Clustering Over Indexes]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/' | relative_url }}) | 6 |
| 7 | [The Procedure Autopsy: Decomposing PL/SQL]({{ '/03-legacy-migration-to-databricks/07-the-procedure-autopsy-decomposing-pl-sql/' | relative_url }}) | 6 |
| 8 | [Pattern Translation: Cursors, Triggers, Temp Tables, MERGE]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/' | relative_url }}) | 7 |
| 9 | [AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/' | relative_url }}) | 7 |
| 10 | [The Ingestion Decision Tree]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/' | relative_url }}) | 6 |
| 11 | [CDC and Lakeflow Declarative Pipelines With Data Contracts]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/' | relative_url }}) | 7 |
| 12 | [The Reconciliation Stack: Proving Semantic Parity]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/' | relative_url }}) | 6 |
| 13 | [Building the Reconciliation Engine]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/' | relative_url }}) | 6 |
| 14 | [The Parallel Run and Zero-Downtime Cutover]({{ '/03-legacy-migration-to-databricks/14-the-parallel-run-and-zero-downtime-cutover/' | relative_url }}) | 7 |
| 15 | [Unity Catalog and the Privilege Migration]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/' | relative_url }}) | 6 |
| 16 | [ABAC, Masking, and Cross-Engine Governance]({{ '/03-legacy-migration-to-databricks/16-abac-masking-and-cross-engine-governance/' | relative_url }}) | 6 |
| 17 | [The Cost Iceberg and Compute Arbitrage]({{ '/03-legacy-migration-to-databricks/17-the-cost-iceberg-and-compute-arbitrage/' | relative_url }}) | 6 |
| 18 | [FinOps Engineering: System Tables and Chargeback]({{ '/03-legacy-migration-to-databricks/18-finops-engineering-system-tables-and-chargeback/' | relative_url }}) | 7 |
| 19 | [The Migration Playbook: Sequencing the Whole Thing]({{ '/03-legacy-migration-to-databricks/19-the-migration-playbook-sequencing-the-whole-thing/' | relative_url }}) | 6 |
| 20 | [The War Room: Capstone Simulation]({{ '/03-legacy-migration-to-databricks/20-the-war-room-capstone-simulation/' | relative_url }}) | 6 |

<!-- prevnext:start -->

---

| [&larr; Previous: Final Stage - Production Deployment and Smoke Testing Your Project]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/final-stage-production-deployment-and-smoke-testing-your-project/' | relative_url }}) | [Next: Why Enterprise Migrations Fail (And the Architect Who Stops It) &rarr;]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

