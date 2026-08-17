---
title: "Implementing Users, Groups, and Access Control"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-users-groups-and-access-control/
read_minutes: 13
---

# Implementing Users, Groups, and Access Control
{: .no_toc }

*Estimated read: 13 min*

With users, groups, and service principals defined conceptually in the last lecture, this one
builds a realistic dev/uat/prod access structure end to end -- the exact shape you'll want for
[Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}).

## Creating groups

Through the account console (**Admin Console -> Identity and access -> Groups**) or via the
Databricks CLI/SDK for automation:

```bash
databricks account groups create --display-name "data-engineering"
databricks account groups create --display-name "uat-reviewers"
```

## Adding members

```bash
databricks account groups add-member \
  --group-id <group-id> \
  --member-id <user-id>
```

Through the console, this is a simple add-to-group action -- worth automating via the CLI/API once
you have more than a handful of people, since the same group membership needs to stay in sync
across account-level and workspace-level access as your team grows.

## A realistic three-environment grant structure

```sql
-- dev: broad access for active development
GRANT USE CATALOG, USE SCHEMA, CREATE SCHEMA, CREATE TABLE, MODIFY, SELECT
ON CATALOG dev
TO `data-engineering`;

-- uat: read plus controlled write, reviewers can validate but not casually modify
GRANT USE CATALOG, USE SCHEMA, SELECT
ON CATALOG uat
TO `uat-reviewers`;

GRANT USE CATALOG, USE SCHEMA, MODIFY, SELECT
ON CATALOG uat
TO `data-engineering`;

-- prod: narrow, mostly service-principal-only write access
GRANT USE CATALOG, USE SCHEMA, SELECT
ON CATALOG prod
TO `data-engineering`;

GRANT USE CATALOG, USE SCHEMA, MODIFY, SELECT
ON CATALOG prod
TO `steprightjob-service-principal`;
```

**Key term:** notice the pattern -- **write access narrows as you move toward production**.
Engineers can read prod for debugging, but the service principal running the actual scheduled job
is the only identity with `MODIFY` there. This is the same discipline a well-governed warehouse
enforced through separate prod-deployment accounts; Unity Catalog just makes the grant explicit and
auditable rather than a tribal-knowledge convention.
{: .important }

## Workspace-level catalog binding

Beyond object-level grants, Unity Catalog also supports **workspace-catalog bindings** --
restricting *which workspaces* a catalog is even visible from, independent of user-level grants:

```sql
ALTER CATALOG prod SET OWNER TO `platform-admins`;
-- Bind the prod catalog to only the prod workspace, via account console workspace bindings
```

This adds a second layer of isolation: even a user with a valid grant on the `prod` catalog can't
query it from a `dev` workspace if that workspace isn't bound to see `prod` at all -- useful for
organizations where "which workspace you're in" should itself constrain what's reachable,
independent of individual permissions.

## Verifying the structure works as intended

```sql
-- As a member of uat-reviewers, this should succeed:
SELECT * FROM uat.sales.orders LIMIT 10;

-- As a member of uat-reviewers, this should fail:
INSERT INTO uat.sales.orders VALUES (...);
```

Testing both the allowed and the denied path is worth doing explicitly rather than assuming a
grant structure works as designed -- the same instinct as testing both a positive and negative case
for a warehouse role's permissions before trusting it in production.

<!-- prevnext:start -->

---

| [&larr; Previous: Identity in Unity Catalog - Users, Groups, and Service Principals]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/identity-in-unity-catalog-users-groups-and-service-principals/' | relative_url }}) | [Next: Unity Catalog Permissions Model - GRANT &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-model-grant/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
