---
title: "Download Project Source Code"
parent: "StepRight - Project Foundation"
grand_parent: "StepRight Capstone Project"
nav_order: 6
permalink: /02-stepright-capstone-project/01-stepright-project-foundation/download-project-source-code/
read_minutes: 2
---

# Download Project Source Code
{: .no_toc }

*Estimated read: 2 min*

This guide doesn't ship a single zip file with the finished `steprightproject` repo inside it --
and deliberately so. Every code sample across Sections 1-8 is written to be copy-paste-ready
directly from the lecture page it appears on, in the order you'd actually write it: the `CREATE
CATALOG`/`CREATE SCHEMA`/`CREATE VOLUME` statements from Lecture 3, the generator script from
Lecture 5, the bronze pipeline files from Section 2, and so on through Section 8's deployment
bundle. Typing or pasting each piece into the Git folder as you reach it builds the same muscle
memory as building a real project incrementally, commit by commit, instead of unzipping a finished
answer and reverse-engineering how it works.

## What to do instead

If you followed Lectures 1 through 5 in order, you already have a working `steprightproject`
skeleton: a Git folder connected to your own private repository, a `dev.step_right` schema, a
seeded landing volume, and a generator script under `scripts/`. That repository -- yours, not a
downloaded copy -- is the actual project you'll extend for the rest of Part 2.

Two habits make that easier as the project grows:

- **Commit after every lecture**, not only at the end of a section. A pipeline file from Section 2
  and its corresponding unit test from Section 6 are much easier to review as two small, related
  commits than as one enormous diff at the end of the capstone.
- **Keep the six-folder skeleton from Lecture 2 intact.** Every later lecture assumes
  `transformations/`, `resources/`, `tests/`, `notebooks/`, `scripts/`, and `databricks.yml` exist
  in their planned locations -- resist the urge to reorganize mid-project, since Section 8's
  Asset Bundle configuration references these paths directly.

With the skeleton in place and batch zero seeded, Section 2 starts writing the first real
pipeline: a bronze layer for the CDC-sourced tables.

<!-- prevnext:start -->

---

| [&larr; Previous: Seed the Project - Data Generator and Loader Notebook]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/seed-the-project-data-generator-and-loader-notebook/' | relative_url }}) | [Next: StepRight - Ingestion Layer &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

