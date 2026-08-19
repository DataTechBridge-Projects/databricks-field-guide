---
title: "Databricks Glossary"
nav_order: 6
permalink: /glossary/
---

# Databricks Glossary

A quick-reference glossary of Databricks-specific terminology used throughout this guide, grouped
the same way the course itself is organized: platform and compute, storage, governance, the
Lakeflow pillars, the newer AI/agentic layer, and the migration-specific vocabulary from Part 3.
Where a legacy-warehouse or ETL equivalent makes a term click faster, it's noted alongside the
definition. This page is a companion to -- not a replacement for -- Databricks' own
[technical terminology glossary](https://docs.databricks.com/aws/en/resources/glossary), which is
the canonical, continuously-updated source if a term you're looking for isn't here.

## Platform and compute

| Term | What it is |
|------|------------|
| **Workspace** | A Databricks deployment tied to a cloud account -- the container for your notebooks, jobs, clusters, and catalog access. The closest legacy analogue is a database instance plus its surrounding tooling, bundled into one environment. |
| **Cluster** | The compute behind a notebook or job -- **all-purpose** (interactive), **jobs** (ephemeral, spun up per run), or **serverless** (no cluster to manage at all). |
| **Photon** | Databricks' native, vectorized query engine that accelerates SQL and DataFrame workloads -- think of it as the equivalent of your legacy engine's cost-based optimizer, but rewritten in C++ for columnar execution. |
| **DBU (Databricks Unit)** | The unit compute is billed in, independent of the underlying cloud instance price -- your FinOps team will care about this more than almost anything else in this list. |
| **Cluster policy** | Admin-defined guardrails on what a cluster can be configured as (instance types, autoscaling limits, max cost) -- the Databricks equivalent of a DBA-enforced resource group or query governor. |
| **Databricks Runtime** | The managed image of Apache Spark plus supporting libraries that a cluster boots from -- versioned, so a job can pin to a specific runtime the way a legacy job might pin to a specific database patch level. |

## Storage and table format

| Term | What it is |
|------|------------|
| **Delta Lake** | The open-source, ACID-compliant [storage layer](https://docs.databricks.com/aws/en/delta/) every table in this course sits on -- Parquet files plus a transaction log, giving you the reliability guarantees a legacy relational engine gave you for free, on top of cheap object storage. |
| **Delta table** (managed vs. external) | A **managed** table's data lifecycle is fully owned by Unity Catalog; an **external** table points at storage you control -- the same distinction as an Oracle tablespace you manage versus an external table pointing at a flat file. |
| **Transaction log** | The ordered record of every change made to a Delta table, which is what makes ACID guarantees and time travel possible. |
| **Time travel** | Querying a Delta table as of a previous version or timestamp -- no separate flashback or point-in-time-recovery feature required. |
| `OPTIMIZE` / `VACUUM` | `OPTIMIZE` compacts small files for read performance; `VACUUM` removes files no longer referenced by the transaction log, past a retention window. |
| `MERGE INTO` | Delta Lake's native upsert statement -- the direct replacement for a legacy `MERGE` or a hand-rolled `UPDATE`-then-`INSERT` pattern. |
| **Schema enforcement / evolution** | Enforcement rejects writes that don't match a table's schema; evolution allows a write to safely widen it (e.g., add a column) instead of failing. |
| **Liquid Clustering** | The modern, adaptive replacement for manual partitioning and Z-Ordering -- Databricks decides and maintains the physical layout instead of you hand-picking partition keys. |
| **Z-Ordering / partitioning** | Older physical-layout techniques, still supported and still relevant for very large or unusually-shaped tables where Liquid Clustering isn't (yet) the default recommendation. |

## Governance (Unity Catalog)

| Term | What it is |
|------|------------|
| **Unity Catalog** | The [unified governance layer](https://docs.databricks.com/aws/en/data-governance/unity-catalog/) for data and AI across a Databricks account -- access control, lineage, and audit logging in one place, replacing per-database grants and disconnected DBA runbooks. |
| **Metastore -> catalog -> schema -> table/volume** | Unity Catalog's object hierarchy -- one metastore per region, catalogs beneath it, schemas beneath those, tables and volumes at the leaf. |
| **External location / storage credential** | The Unity Catalog-side pointer and permission object for a cloud storage path -- the equivalent of an Oracle directory object plus the OS-level grant your DBA used to hand out separately. |
| **Volume** | A Unity Catalog-governed pointer to non-tabular files (images, PDFs, model artifacts) living in cloud storage. |
| **Service principal** | A non-human identity used for job and application authentication -- the Databricks equivalent of a service account. |
| **ABAC** | Attribute-based access control -- granting access based on tags/attributes rather than one-off per-object grants, which scales far better across a large catalog. |
| **Dynamic view** | A view whose row/column visibility changes based on who's querying it -- how row-level security and column masking are typically implemented. |
| **Delta Sharing** | An open protocol for sharing live Delta tables across organizations or platforms without copying data. |
| **Credential vending** | Unity Catalog issuing short-lived, scoped cloud credentials to a query at runtime, instead of a long-lived key sitting in a config file. |
| **Glossary & Domains** *(new)* | A shared business-terminology and domain-ownership layer inside Unity Catalog, so people and AI agents resolve terms like "active customer" the same way. |
| **Business Semantics / Metric Views** *(new, GA 2026)* | Databricks' [native semantic layer](https://docs.databricks.com/aws/en/business-semantics/) -- metric views separate measure definitions from the dimensions used to group them, so every report of a given KPI across the org agrees, the same job a curated semantic layer or a set of "golden" reporting views used to do in a legacy BI stack. |

## Architecture and Lakeflow (ingestion, transformation, orchestration)

| Term | What it is |
|------|------------|
| **Medallion architecture** | The bronze (raw) / silver (cleaned, conformed) / gold (business-level aggregate) layering pattern this course uses throughout. |
| **Lakehouse** | The architecture combining data-lake storage economics with data-warehouse transactional and governance guarantees -- the reason a Databricks migration isn't "just move the tables to S3." |
| **Lakeflow** | The umbrella brand for Databricks' three data-engineering pillars -- ingestion, transformation, and orchestration -- all governed by Unity Catalog. |
| **Lakeflow Connect** | The [ingestion pillar](https://docs.databricks.com/aws/en/ingestion/overview) -- 100+ managed connectors for databases, SaaS apps, files, and message buses, replacing a hand-built Talend-style extraction job per source. |
| **Lakeflow Declarative Pipelines / SDP** | The [transformation pillar](https://docs.databricks.com/aws/en/ldp) (formerly Delta Live Tables) -- flows, streaming tables, materialized views, and `AUTO CDC`, where you declare the target shape of the data and Databricks manages the incremental execution. |
| **Lakeflow Jobs** | The [orchestration pillar](https://docs.databricks.com/aws/en/jobs) -- tasks, DAGs, parameters, retries, and backfill, the direct successor to a legacy scheduler like `DBMS_SCHEDULER` or a cron-driven ETL controller. |
| **Lakeflow Designer** *(new)* | A [visual, no-code canvas](https://docs.databricks.com/aws/en/designer/what-is-lakeflow-designer) for building pipelines by dragging and connecting operators -- still emits governed, production pipeline code underneath, not a black box. |
| **Zerobus Ingest** *(new)* | High-volume, low-latency streaming ingestion for event data, sitting alongside Lakeflow Connect for the highest-throughput sources. |
| **Real-Time Mode** *(new)* | A millisecond-latency execution mode for Declarative Pipelines, for cases where the standard micro-batch cadence isn't fast enough. |
| **Auto Loader** | Incremental, exactly-once file ingestion from cloud storage, the underpinning of most Lakeflow Connect and Declarative Pipeline ingestion. |
| **CDC** | Change data capture -- propagating only the rows that changed at a source, rather than re-extracting a full table every run. |
| **SCD Type 2** | The slowly-changing-dimension pattern that keeps full row history via effective-dated versions -- implemented natively via `AUTO CDC` rather than hand-written `MERGE` logic. |

## AI and agentic layer *(mostly new for 2026)*

| Term | What it is |
|------|------------|
| **Genie** | Databricks' [natural-language data assistant](https://docs.databricks.com/aws/en/genie/) -- ask a business question in plain English, get a governed answer sourced from Unity Catalog tables. Now split into **Genie Ontology** (the semantic model it reasons over) and **Genie One** (the GA'd assistant across web, iOS, and Android). |
| **Agent Bricks** | Databricks' [enterprise agent platform](https://developers.databricks.com/docs/agents/overview) -- unifies model access, execution, governance, and business context for building, evaluating, and deploying production agents, with support for external frameworks (LangGraph, CrewAI, the Claude Code SDK) alongside Databricks-native tooling. |
| **Mosaic AI** | The umbrella for Databricks' model-serving, fine-tuning, and agent-evaluation tooling underneath Agent Bricks. |
| **Lakebase** | A fully [managed, serverless Postgres](https://docs.databricks.com/aws/en/oltp/) database built into the lakehouse -- the operational (OLTP) counterpart to Delta Lake's analytical (OLAP) storage, so an agent or app can read and write transactional data without standing up a separate database. |

## Migration-specific vocabulary (Part 3 of this course)

| Term | What it is |
|------|------------|
| **Lakebridge** | Databricks' migration tooling for schema and code translation from legacy platforms (Oracle, Teradata, SQL Server) onto the Lakehouse. |
| **Rehost / Re-platform / Re-architect** (the 3-R decision) | The three strategies for moving a given workload, in increasing order of effort and long-term payoff -- covered in depth in Part 3, Section 3. |
| **TCO** | Total cost of ownership -- the multi-year cost comparison that turns a migration proposal into something a CFO signs off on. |
| **Workload inventory** | The catalogued list of every table, procedure, job, and report in the legacy estate, scored for migration complexity and priority. |
| **Lakehouse Federation** | Querying a live legacy source (Oracle, SQL Server, etc.) directly from Databricks without first copying the data -- lets a migration start proving value before every table has physically moved. |
| **DDL/schema translation** | Converting legacy `CREATE TABLE` and type definitions into Delta-native equivalents, including the physical-design decisions (Liquid Clustering vs. legacy indexes) that don't map one-to-one. |
| **PL/SQL-to-PySpark transpilation** | Converting procedural legacy code into set-based PySpark/SQL -- almost never a mechanical, one-to-one translation, which is why this course spends a full section on decomposing procedures by hand first. |
| **Reconciliation** (count / sum / checksum / hash / semantic parity) | The five-layer verification stack, covered in Part 3 Sections 12-13, that proves a migrated table or pipeline produces the same results as the legacy source. |
| **Parallel run** | Running old and new systems side by side against live data before cutover, so reconciliation has something continuous to check. |
| **Cutover** (phased vs. big-bang) | The act of switching production traffic from the legacy system to Databricks -- either table-by-table/workload-by-workload (phased) or all at once (big-bang). |
| **Rollback window** | The pre-agreed time period after cutover during which reverting to the legacy system is still possible, and the decision criteria for pulling that trigger. |
| **FinOps** | Databricks cost governance in production -- `system.billing.usage`, chargeback to business units, and predictive optimization, covered in Part 3 Section 18. |

## Databricks Labs (community projects, not officially supported)

A handful of terms from this space come up in the ecosystem but aren't first-party platform
features -- worth recognizing, not worth quizzing on. These live under the `databrickslabs`
GitHub organization, are provided as-is with no SLA, and occasionally graduate into first-party
tooling (Lakebridge itself started this way).

| Term | What it is |
|------|------------|
| **Ontos** | An open-source business catalog for Unity Catalog -- lighter-weight than a full semantic layer. |
| **dqx** | A data-quality checking library for PySpark batch and streaming workloads. |
| **dbldatagen** | A synthetic data generator for populating test and POC environments. |
| **Kasal** | A low-code builder for deploying AI agents on Databricks, predating the broader Agent Bricks platform. |

<!-- prevnext:start -->

---

| [&larr; Previous: Check Your Knowledge]({{ '/03-legacy-migration-to-databricks/20-the-war-room-capstone-simulation/check-your-knowledge/' | relative_url }}) |  |
|:---|---:|

<!-- prevnext:end -->
