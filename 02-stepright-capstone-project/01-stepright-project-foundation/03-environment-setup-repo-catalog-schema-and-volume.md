---
title: "Environment Setup - Repo, Catalog, Schema, and Volume"
parent: "StepRight - Project Foundation"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/01-stepright-project-foundation/environment-setup-repo-catalog-schema-and-volume/
read_minutes: 9
---

# Environment Setup - Repo, Catalog, Schema, and Volume
{: .no_toc }

*Estimated read: 9 min*

Lecture 2 planned the shape of the project; this lecture builds it -- the Git folder, the catalog,
the schema, and the volume `steprightproject` actually depends on, in the order each piece needs
to exist.

## Step 1: create and connect the private Git repo

Create a new, empty, **private** repository named `steprightproject` on GitHub (or your Git
provider of choice) -- empty, so the Git folder controls the initial commit rather than fighting
a pre-populated `README`. Then, in the Databricks workspace:

1. **Workspace -> Create -> Git folder.**
2. Paste the repository's clone URL and confirm the branch (`main`).
3. Confirm the six top-level folders from Lecture 2 (`transformations/`, `resources/`, `tests/`,
   `notebooks/`, `scripts/`) plus `databricks.yml` -- create them now, even empty, with a `.gitkeep`
   placeholder file in each so Git tracks the empty directory.

Every notebook, pipeline file, and bundle resource for the rest of Part 2 gets created inside this
Git folder, not in a personal workspace folder outside version control.

## Step 2: create the catalog and schema

```sql
-- Run as a user/service principal with CREATE CATALOG privilege on the metastore
CREATE CATALOG IF NOT EXISTS dev
  COMMENT 'Development catalog for StepRight capstone project';

CREATE SCHEMA IF NOT EXISTS dev.step_right
  COMMENT 'StepRight bronze, silver, and gold tables';
```

If your workspace's Unity Catalog metastore already has a shared `dev` catalog from earlier
experimentation, reuse it -- the schema, not the catalog, is what isolates this project from
everything else in the workspace. `uat` and `prod` catalogs don't need to exist yet; Section 8's
deployment lecture creates them as part of the bundle promotion path.

## Step 3: create the landing and staging volumes

```sql
CREATE VOLUME IF NOT EXISTS dev.step_right.landing
  COMMENT 'Governed landing zone for StepRight file-based sources';

CREATE VOLUME IF NOT EXISTS dev.step_right.staging
  COMMENT 'Upload target for locally generated test data batches, ahead of loading into landing';
```

Managed volumes are enough here -- there's no external cloud storage bucket to point at, since
Databricks manages the underlying storage for a managed volume the same way it does for a managed
Delta table. `landing` is what Section 2's bronze pipelines read from; `staging` is a separate
drop point Lecture 5's loader notebook reads from and copies into `landing`, standing in for
however files would first arrive from outside Databricks (SFTP, a cloud storage event, a partner
upload) before a pipeline expects to see them. Create the four source-specific subfolders Lecture
2 planned inside `landing` by writing an empty placeholder file into each with `dbutils.fs`, since
empty directories aren't addressable in volumes until something exists inside them:

```python
subfolders = ["products", "inventory", "clickstream", "fulfillment"]
for name in subfolders:
    path = f"/Volumes/dev/step_right/landing/{name}/.gitkeep"
    dbutils.fs.put(path, "", overwrite=True)
```

## Step 4: verify

```sql
SHOW SCHEMAS IN dev;
SHOW VOLUMES IN dev.step_right;
LIST '/Volumes/dev/step_right/landing/';
```

You should see `step_right` in the schema list, both `landing` and `staging` in the volume list,
and all four subfolders under `landing` present with their placeholder files.

Get catalog and schema names wrong here and every downstream reference -- pipeline definitions in
Section 2, test fixtures in Section 6, bundle resources in Section 8 -- inherits the mistake.
`dev.step_right` is the name used consistently for the rest of Part 2; if you choose a different
catalog or schema name, substitute it everywhere a later lecture references `dev.step_right`.
{: .important }

## Optional: a CDC staging schema

The CDC-sourced tables (`orders`, `order_items`, `customers`) land through a Lakeflow Connect
connector into a raw schema, separate from `step_right` itself, matching the pattern from Part 1,
Section 8:

```sql
CREATE SCHEMA IF NOT EXISTS dev.raw_cdc
  COMMENT 'Raw CDC change feed landing schema, read by Section 2 bronze pipelines';
```

Section 2 configures the actual connector against this schema once the bronze pipeline design is
in place -- for now, having the schema exist is enough to unblock that work.

<!-- prevnext:start -->

---

| [&larr; Previous: Project Structure - Planning the Initial Project Structure]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/project-structure-planning-the-initial-project-structure/' | relative_url }}) | [Next: Test Data Strategy - Planning your Test Data Preparation &rarr;]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/test-data-strategy-planning-your-test-data-preparation/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

