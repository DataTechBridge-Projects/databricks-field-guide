---
title: "Databricks Fundamentals"
nav_order: 3
has_children: true
permalink: /01-databricks-fundamentals/
---

# Databricks Fundamentals

Core Databricks and Apache Spark concepts for a data engineer coming from a legacy warehouse or ETL background: workspace, Delta Lake, Unity Catalog, Medallion architecture, and the Lakeflow ingestion, transformation, and orchestration pillars.

If you're coming from a legacy warehouse or a hand-built ETL tool, this part is where the
Databricks-native vocabulary gets mapped onto what you already know: a **Delta table** instead of
a heap table with a transaction log bolted on by convention, **Unity Catalog** instead of
GRANT statements scattered across a dozen schemas, and the three **Lakeflow** pillars
(Connect, Declarative Pipelines, Jobs) instead of a Talend job, a stored procedure, and a cron
entry stitched together by hand. By Section 10 you'll have a working mental model of the whole
platform -- ready to build something real in [Part 2]({{ '/02-stepright-capstone-project/' | relative_url }}).

## Sections

| # | Section | Items |
|---|---------|-------|
| 1 | [Before you start]({{ '/01-databricks-fundamentals/01-before-you-start/' | relative_url }}) | 3 |
| 2 | [Introduction]({{ '/01-databricks-fundamentals/02-introduction/' | relative_url }}) | 6 |
| 3 | [Sign up for Databricks Platform in AWS]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/' | relative_url }}) | 8 |
| 4 | [Working in Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }}) | 8 |
| 5 | [Delta Lake - Deep Dive]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/' | relative_url }}) | 12 |
| 6 | [Unity Catalog - Governance from day one]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/' | relative_url }}) | 9 |
| 7 | [Medallion Architecture - Design and Implementation]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/' | relative_url }}) | 6 |
| 8 | [Lakeflow Connect - Ingestion Pillar]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/' | relative_url }}) | 8 |
| 9 | [Lakeflow Spark Declarative Pipelines - Transformation Pillar]({{ '/01-databricks-fundamentals/09-lakeflow-spark-declarative-pipelines-transformation-pillar/' | relative_url }}) | 8 |
| 10 | [Lakeflow Jobs - The Orchestration Pillar]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/' | relative_url }}) | 8 |

<!-- prevnext:start -->

---

| [&larr; Previous: Course Map]({{ '/course-map/' | relative_url }}) | [Next: Before you start &rarr;]({{ '/01-databricks-fundamentals/01-before-you-start/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

