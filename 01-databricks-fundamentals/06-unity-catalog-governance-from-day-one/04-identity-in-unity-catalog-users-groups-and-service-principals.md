---
title: "Identity in Unity Catalog - Users, Groups, and Service Principals"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 4
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/identity-in-unity-catalog-users-groups-and-service-principals/
read_minutes: 12
---

# Identity in Unity Catalog - Users, Groups, and Service Principals
{: .no_toc }

*Estimated read: 12 min*

Before granting any permission, Unity Catalog needs to know *who* it's granting it to. Three kinds
of identity cover essentially every case: **users**, **groups**, and **service principals** -- the
last being the one most people coming from a warehouse background haven't used by this exact name
before.

## Users

An individual human, tied to an email address, added to the account either manually or (in a real
organization) synced from an identity provider (Okta, Azure AD, etc.) via SCIM. For this guide's
learning purposes, users are added manually through the account console -- production setups
almost always automate this via identity provider sync instead.

## Groups

```sql
GRANT USE CATALOG, SELECT ON CATALOG sales TO `data-engineering-team`;
```

**Groups** are the unit you should actually grant permissions to, not individual users -- the same
discipline a well-run warehouse enforced through roles. Add and remove users from a group as team
membership changes; the underlying grants never need to be touched. Granting directly to
individuals is the Unity Catalog equivalent of granting warehouse permissions to a personal login
instead of a role -- it works, and it's exactly the kind of thing that becomes unmaintainable at
scale.
{: .important }

## Service principals: identity for automation

**Key term:** a **service principal** is a non-human identity -- for a job, a pipeline, a CI/CD
process -- that needs its own credentials and permissions, entirely separate from any individual
person's account. This is the direct equivalent of a "service account" you'd have created for a
legacy ETL tool's database connection, except Unity Catalog treats it as a first-class identity
that can own objects, hold grants, and appear in audit logs on equal footing with a human user.
{: .important }

```sql
GRANT USE CATALOG, USE SCHEMA, MODIFY, SELECT
ON SCHEMA prod.sales
TO `steprightjob-service-principal`;
```

Why this matters specifically: a scheduled Lakeflow Job (Section 10) should run **as a service
principal**, not as whichever human happened to create the job. If that person leaves the
organization or their personal credentials are deactivated, a job running under their identity
breaks -- a service principal has no such dependency on any one person's employment status.

## Practical identity design for this guide

| Identity type | Used for |
|---|---|
| Individual users | Interactive development, ad hoc queries, debugging |
| Groups (`data-engineering`, `uat-team`, etc.) | All permission grants -- never grant directly to a user |
| Service principals | Scheduled jobs, pipelines, CI/CD automation |

This three-tier model recurs directly in
[Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}) -- its production deployment section
runs the final orchestration job under a dedicated service principal, not a developer's personal
account, exactly the pattern established here.

## Checking who has access to what

```sql
SHOW GRANTS ON CATALOG sales;
SHOW GRANTS TO `data-engineering-team`;
```

Both directions matter: "what can this catalog be accessed by" and "what can this group access"
-- the second is what you'd check before removing a group's membership from a team, to understand
exactly what access changes.

<!-- prevnext:start -->

---

| [&larr; Previous: Catalogs, External Locations and Storage Credentials]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/catalogs-external-locations-and-storage-credentials/' | relative_url }}) | [Next: Implementing Users, Groups, and Access Control &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-users-groups-and-access-control/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
