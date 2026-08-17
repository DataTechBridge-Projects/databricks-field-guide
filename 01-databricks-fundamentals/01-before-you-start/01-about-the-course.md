---
title: "About the Course"
parent: "Before you start"
grand_parent: "Databricks Fundamentals"
nav_order: 1
permalink: /01-databricks-fundamentals/01-before-you-start/about-the-course/
read_minutes: 3
---

# About the Course
{: .no_toc }

*Estimated read: 3 min*

**The Complete Databricks Field Guide** is written for one specific reader: a data engineer who
has already spent real time running a legacy enterprise warehouse -- Oracle, Teradata, or SQL
Server -- or hand-building pipelines in a tool like **Talend**, and who now needs to get fluent on
**Databricks** without re-learning data engineering from scratch. You already know what a fact
table is, what an incremental load protects you from, and why a 2 a.m. batch job failure is
someone's problem. This guide maps that experience onto Databricks-native concepts instead of
re-teaching the fundamentals of ETL you already own.

The site is organized into three parts, meant to be read in order:

| Part | What it covers |
|------|-----------------|
| 1. **Databricks Fundamentals** | Workspace, clusters, Delta Lake, Unity Catalog, Medallion architecture, and the three Lakeflow pillars (Connect, Declarative Pipelines, Jobs). |
| 2. **StepRight Capstone Project** | A full, hands-on data engineering project built end-to-end across ingestion, transformation, gold-layer reporting, orchestration, unit testing, data quality monitoring, and CI/CD. |
| 3. **Legacy Migration to Databricks** | The specialist track: assessing a legacy EDW, translating schemas and stored procedures, proving reconciliation, executing cutover, and migrating governance and cost management onto the Lakehouse. |

Part 1 gives you the vocabulary and the platform mental model. Part 2 forces you to apply it by
building something real, the way a tutorial with only isolated snippets never does. Part 3 is
where the legacy-warehouse experience you're bringing to this becomes a direct asset rather than
something to unlearn -- it is written specifically for the migration architect role.

**Key term:** throughout this guide, **Lakehouse** refers to Databricks' combination of a data
lake's low-cost, open storage with a data warehouse's transactional guarantees and governance --
the thing that makes "one platform for the fact table and the raw JSON dump" possible.

## How this guide is different from a video course

There is no video here. Every page is a written article you can skim, search, and copy code out
of -- which also means you can read faster than any narrator would talk. Each lecture page shows
an **estimated read time** instead of a video duration.
{: .important }

If you want the full syllabus at a glance before committing to a reading order, start with the
[Course Map]({{ '/course-map/' | relative_url }}) -- it lists every part, section, and lecture in one place, deliberately
set in a different typeface so it reads as a map, not a lesson.

## What you'll be able to do by the end

By the end of Part 1 you'll be comfortable running production-shaped Delta Lake pipelines with
proper governance. By the end of Part 2 you'll have built a complete project -- StepRight -- with
tests, CI/CD, and monitoring, not just followed along. By the end of Part 3 you'll be able to walk
into a legacy EDW migration, profile the real workload, defend a TCO number to a CFO, and run a
cutover without losing anyone's trust in the numbers.

Continue to [Course Prerequisites]({{ '/01-databricks-fundamentals/01-before-you-start/course-prerequisites/' | relative_url }})
to confirm what you need before you start.

<!-- prevnext:start -->

---

| [&larr; Previous: Before you start]({{ '/01-databricks-fundamentals/01-before-you-start/' | relative_url }}) | [Next: Course Prerequisites &rarr;]({{ '/01-databricks-fundamentals/01-before-you-start/course-prerequisites/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
