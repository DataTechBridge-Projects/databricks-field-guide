---
title: "Implementing Unity Catalog Permission for Catalog and Schema"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-unity-catalog-permission-for-catalog-and-schema/
read_minutes: 12
---

# Implementing Unity Catalog Permission for Catalog and Schema
{: .no_toc }

*Estimated read: 12 min*

A worked, end-to-end example: two groups, one catalog, deliberately scoped so one group can create
schemas and the other explicitly cannot -- and neither can touch the metastore or external
locations directly. This is the pattern to copy for a real project's initial access setup.

## The scenario

- **`data-engineering`** group: needs to create new schemas as the project grows, and full
  read/write within them.
- **`uat-team`** group: needs to read data for validation, but should never create schemas or
  modify existing tables -- a UAT reviewer shouldn't be able to accidentally change what they're
  validating.
- Neither group should be able to create or modify **external locations** or **storage
  credentials** -- that stays with a small platform-admin group.

## Granting `data-engineering`

```sql
GRANT USE CATALOG ON CATALOG dev TO `data-engineering`;
GRANT CREATE SCHEMA ON CATALOG dev TO `data-engineering`;
GRANT USE SCHEMA, CREATE TABLE, MODIFY, SELECT
ON SCHEMA dev.sales
TO `data-engineering`;
```

`CREATE SCHEMA` at the catalog level, plus full working privileges scoped to the schemas that
already exist -- this group can grow the catalog's structure and work fully within it.

## Granting `uat-team` (deliberately narrower)

```sql
GRANT USE CATALOG ON CATALOG dev TO `uat-team`;
GRANT USE SCHEMA, SELECT
ON SCHEMA dev.sales
TO `uat-team`;
-- Deliberately omitted: CREATE SCHEMA, CREATE TABLE, MODIFY
```

`uat-team` can navigate to and read from `dev.sales`, and nothing more. No `CREATE SCHEMA` means
they can't spin up new, ungoverned schemas. No `MODIFY` means they can't alter the data they're
reviewing, even accidentally.

## Confirming what was denied, not just what was granted

```sql
-- As uat-team, this should succeed
SELECT * FROM dev.sales.orders LIMIT 5;

-- As uat-team, this should fail with an access-denied error
INSERT INTO dev.sales.orders VALUES (...);

-- As uat-team, this should fail -- no CREATE SCHEMA privilege
CREATE SCHEMA dev.marketing;
```

**Key term:** testing the **negative case** -- confirming an operation is actually denied, not
just that the intended operation succeeds -- is the step people skip under time pressure and
regret later. A permission structure that's never had its denials tested is a permission structure
you're assuming works, not one you've verified.
{: .important }

## Metastore admin and external locations stay separate

```sql
-- Neither data-engineering nor uat-team receive this:
GRANT CREATE EXTERNAL LOCATION ON METASTORE dev_metastore TO `platform-admins`;
```

Keeping storage-credential and external-location creation restricted to a small platform-admin
group -- separate from the day-to-day data-engineering group -- means the group building pipelines
never holds the keys to raw cloud storage access directly, only to the governed catalog objects
built on top of it. This mirrors the separation-of-duties principle a well-run warehouse team
already applies between "who can create a linked server" and "who can query through one."

## The result

Two groups, cleanly separated by what they're actually allowed to do, verified both positively and
negatively, with the storage-credential layer kept entirely out of either group's reach. This
exact pattern -- environment-scoped catalogs, domain-scoped schemas, narrow write access, verified
denials -- is what [Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}) assumes is already
in place before its pipelines are built.

<!-- prevnext:start -->

---

| [&larr; Previous: Unity Catalog Permissions Model - GRANT]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-model-grant/' | relative_url }}) | [Next: Unity Catalog Permissions for Tables and Volumns &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-for-tables-and-volumns/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
