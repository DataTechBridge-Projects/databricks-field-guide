---
title: "Project Structure - Planning the Initial Project Structure"
parent: "StepRight - Project Foundation"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/01-stepright-project-foundation/project-structure-planning-the-initial-project-structure/
read_minutes: 6
---

# Project Structure - Planning the Initial Project Structure
{: .no_toc }

*Estimated read: 6 min*

Before writing a line of pipeline code, StepRight's project needs a home -- a repo, a place in
Unity Catalog, and a folder structure that won't need restructuring the moment Section 2 shows up.
This lecture plans both.

## Connect Databricks to GitHub first

Every code file this project produces should exist in version control from the first commit, not
retrofitted once the project "feels real." In the Databricks workspace, that means creating a
**Git folder** -- **Workspace -> Create -> Git folder** -- backed by a new, private GitHub
repository named `steprightproject`. A Git folder behaves like a working directory git already
knows about: commits, branches, and pull requests happen through the same Git provider panel
you'd use for any other repo, but every file inside it lives inside the Databricks workspace where
notebooks, pipeline definitions, and bundle configs can reference each other with relative paths.

This is a deliberate departure from how a lot of legacy ETL work gets built -- a Talend job
exported once as a `.zip` and manually copied between environments has no history and no diff. A
Git folder connected to a real repo means every pipeline change in Sections 2-8 is reviewable and
revertible from day one.

## The six-step project skeleton

`steprightproject` is organized around the same six concerns the rest of Part 2 builds out, one
folder per concern:

```text
steprightproject/
├── transformations/      # Bronze / silver / gold pipeline code (Sections 2-4)
├── resources/             # Job and pipeline definitions, Asset Bundle resources (Section 5, 8)
├── tests/                 # Unit and integration tests (Section 6, 8)
├── notebooks/             # One-off setup, seeding, and exploration notebooks (this section)
├── scripts/               # The Faker data generator and other local tooling (Section 1.5)
└── databricks.yml         # Asset Bundle root config tying it all together (Section 8)
```

Six folders map directly to six later sections -- that's not a coincidence, it's the point. Rather
than guessing at structure as the project grows, every folder that will hold real content already
has a name and a reason to exist before Section 2 writes its first pipeline file.

| Folder | Fills in during |
|---|---|
| `transformations/` | Section 2 (bronze), Section 3 (silver), Section 4 (gold) |
| `resources/` | Section 5 (job definition), Section 8 (bundle resources) |
| `tests/` | Section 6 (unit tests), Section 8 (integration tests) |
| `notebooks/` | Section 1 (this section's loader notebook) |
| `scripts/` | Section 1 (the Faker generator, Lecture 5) |
| `databricks.yml` | Section 8 (packaging and deployment) |

## Where the data lives: catalog, schema, volume

Code lives in the Git folder; data lives in Unity Catalog. StepRight's tables and volumes all sit
under one schema: **`dev.step_right`** -- a `dev` catalog (with `uat` and `prod` counterparts
introduced in Section 8's deployment lecture) holding a single `step_right` schema for every
bronze, silver, and gold table this project produces.

Inside that schema, one **volume** -- `dev.step_right.landing` -- holds the source-specific
subfolders every file-based bronze pipeline in Section 2 reads from:

```text
/Volumes/dev/step_right/landing/
├── products/
├── inventory/
├── clickstream/
└── fulfillment/
```

CDC-sourced tables (`orders`, `order_items`, `customers`) don't need a landing subfolder -- Lakeflow
Connect writes their raw change feed directly into a governed table, the same pattern from Part 1,
Section 8. Only the batch file sources need a place to land before Auto Loader picks them up.

A **landing volume** is a Unity Catalog volume used as a governed drop zone for raw files before
any pipeline reads them -- the Databricks-native replacement for the shared network drive or SFTP
staging folder a legacy ETL job used to poll, with access control, lineage, and audit logging
built in rather than bolted on.

Lecture 3 turns this plan into actual `CREATE CATALOG` / `CREATE SCHEMA` / `CREATE VOLUME`
statements; Lecture 5 puts real files into it.

<!-- prevnext:start -->

---

| [&larr; Previous: What are we building-StepRight Architecture Walkthrough]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/what-are-we-building-stepright-architecture-walkthrough/' | relative_url }}) | [Next: Environment Setup - Repo, Catalog, Schema, and Volume &rarr;]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/environment-setup-repo-catalog-schema-and-volume/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

