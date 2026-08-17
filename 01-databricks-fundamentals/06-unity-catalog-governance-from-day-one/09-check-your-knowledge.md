---
title: "Check Your Knowledge"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 9
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- Unity Catalog's architecture, storage credentials
and external locations, identity, and the GRANT permissions model.

1. What is the three-level namespace used to address any Unity Catalog table?
   A. `workspace.schema.table`
   B. `catalog.schema.object`
   C. `metastore.catalog.table`
   D. `account.workspace.table`

2. What is the key advantage of a metastore spanning multiple workspaces?
   A. It reduces storage costs
   B. The same governed tables, grants, and audit log are consistent across dev/uat/prod workspaces rather than duplicated per environment
   C. It removes the need for IAM roles entirely
   D. It allows unlimited free compute

3. What does a **storage credential** represent?
   A. A user's login password
   B. An authentication mechanism tying Unity Catalog to a specific IAM role for accessing cloud storage
   C. A Databricks Runtime version
   D. A cluster policy

4. Why should permissions generally be granted to **groups** rather than individual users?
   A. Groups are required by Unity Catalog syntax
   B. Team membership changes without needing to touch the underlying grants
   C. Individual grants are not supported
   D. Groups grant broader access automatically

5. What is a **service principal** used for?
   A. Interactive human login only
   B. A non-human identity for jobs, pipelines, and automation, independent of any individual person's account
   C. A type of storage credential
   D. A row-level security policy

6. Why is `USE CATALOG` considered a prerequisite grant rather than a standalone useful one?
   A. It has no actual effect
   B. Without it, an identity cannot navigate to objects beneath the catalog even with SELECT granted on them
   C. It automatically grants SELECT on all tables
   D. It only applies to service principals

7. What happens to privilege grants when a catalog's ownership is transferred from an individual to a group?
   A. All existing grants are deleted
   B. Ownership moves to the group, avoiding an ownership tied to one person who may later leave
   C. The catalog becomes read-only
   D. Nothing -- ownership cannot be transferred

8. What do `READ VOLUME` and `WRITE VOLUME` privileges govern?
   A. Access to Delta table rows
   B. Access to a volume's underlying files, without requiring separate S3-level IAM grants for individual users
   C. Cluster compute allocation
   D. Job scheduling permissions

9. What do row filters and column masks enable in Unity Catalog?
   A. Faster query execution
   B. Restricting which rows are returned or masking specific column values based on the querying identity
   C. Automatic schema evolution
   D. Cross-catalog table replication

10. In the environment-scoped grant pattern from this section, why does write access typically narrow toward production?
    A. Production clusters are slower
    B. To limit who can modify production data, often to a service principal running the scheduled job rather than individual engineers
    C. Production catalogs don't support MODIFY grants
    D. It's a Unity Catalog technical limitation, not a design choice

## Answer Key

1. **B** -- `catalog.schema.object` is Unity Catalog's three-level namespace.
2. **B** -- one metastore serving multiple workspaces keeps governed tables, grants, and audit logs consistent everywhere.
3. **B** -- a storage credential authenticates Unity Catalog to a specific cloud IAM role.
4. **B** -- granting to groups means membership changes don't require touching the underlying grants.
5. **B** -- service principals are non-human identities for automation, independent of any individual account.
6. **B** -- `USE CATALOG`/`USE SCHEMA` are prerequisites; without them, deeper grants like SELECT have no effective path.
7. **B** -- ownership transfers to the group, removing the dependency on one specific individual.
8. **B** -- volume privileges govern file-level access without needing direct S3 IAM grants.
9. **B** -- row filters and column masks restrict/redact data based on who's querying.
10. **B** -- narrowing write access toward production limits who (or what automation) can modify production data.

<!-- prevnext:start -->

---

| [&larr; Previous: Unity Catalog Permissions for Tables and Volumns]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-for-tables-and-volumns/' | relative_url }}) | [Next: Medallion Architecture - Design and Implementation &rarr;]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

