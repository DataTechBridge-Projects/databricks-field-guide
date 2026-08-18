---
title: "Check Your Knowledge"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of Unity Catalog architecture and the privilege migration before moving on to ABAC and cross-engine governance.

1. What is the correct three-level namespace order in Unity Catalog?
   A. schema.catalog.table
   B. catalog.schema.table
   C. metastore.catalog.table
   D. table.schema.catalog

2. In Unity Catalog's hierarchy, which object is attached to one or more workspaces and holds the shared identity federation and audit log for a region?
   A. Catalog
   B. Schema
   C. Metastore
   D. External location

3. What is a Unity Catalog volume the closest analog to in a legacy Oracle environment?
   A. A tablespace
   B. A directory object or shared file share for governed non-tabular files
   C. A materialized view
   D. A synonym

4. In the "security layer cake," which layer determines whether a service principal can bypass Unity Catalog entirely by reading Parquet files directly from cloud storage?
   A. Unity Catalog grants
   B. Row-level security
   C. Column masking
   D. Cloud IAM (storage credentials and external locations)

5. What is the direct Unity Catalog successor to an Oracle VPD policy that filtered rows by region?
   A. A column mask
   B. A row filter
   C. A dynamic view
   D. A storage credential

6. When building a privilege migration matrix from an Oracle export, which two Oracle data dictionary views are the primary inventory sources?
   A. DBA_USERS and DBA_PROFILES
   B. DBA_TAB_PRIVS and DBA_ROLE_PRIVS
   C. DBA_OBJECTS and DBA_SEGMENTS
   D. DBA_SYNONYMS and DBA_SEQUENCES

7. What is the main benefit of collapsing 40 per-table grants into a single schema-level grant during privilege migration?
   A. It hides the grant from audit queries
   B. It automatically covers current and future tables in the schema with one statement
   C. It removes the need for any group membership
   D. It bypasses Unity Catalog's ownership model

8. Why do tags help compress "500 roles" down to a much smaller number of groups plus tags?
   A. Tags replace the need for any grants at all
   B. Tags separate "who someone is" (groups) from "what the data is" (sensitivity/domain), avoiding one role per combination
   C. Tags are only used for billing, not access control
   D. Tags automatically delete unused Oracle roles

9. What is the biggest risk of scripting a one-to-one replication of every Oracle role into a Unity Catalog group?
   A. Unity Catalog does not support groups
   B. It permanently imports grant sprawl, nesting complexity, and dead grants into the new system
   C. It is technically impossible due to namespace limits
   D. It requires disabling Unity Catalog's audit log

10. Which outcome is the strongest signal that a privilege migration defaulted to verbatim role replication instead of a matrix-driven redesign?
    A. The number of post-migration groups is within 10% of the original Oracle role count
    B. Every table has a row filter applied
    C. The metastore is attached to more than one workspace
    D. Tags were created before any groups existed

## Answer Key

1. **B** -- Unity Catalog's namespace order is catalog.schema.table, one level more than Oracle's schema.table.
2. **C** -- The metastore is the top-level container attached to workspaces, holding shared identity and audit data.
3. **B** -- A volume is a governed pointer to non-tabular files, playing the role a directory object or shared drive used to play.
4. **D** -- Cloud IAM (storage credentials/external locations) governs raw storage access beneath Unity Catalog's own grant layer.
5. **B** -- A row filter is the Unity Catalog function bound to a table that transparently restricts visible rows, succeeding Oracle VPD.
6. **B** -- DBA_TAB_PRIVS enumerates object-level grants and DBA_ROLE_PRIVS enumerates role membership, the two core inventory sources.
7. **B** -- A schema-level grant applies to every table in the schema, present and future, replacing dozens of per-table GRANTs.
8. **B** -- Tags let a small set of groups (identity) combine with a small set of tags (data sensitivity/domain) instead of multiplying into one role per combination.
9. **B** -- Verbatim replication imports the legacy system's accumulated sprawl, nested hierarchies, and dead grants rather than fixing them.
10. **A** -- Little to no compression in group count versus the original role count indicates no real redesign happened.

<!-- prevnext:start -->

---

| [&larr; Previous: Anti-Pattern: Replicating Oracle's Role Hierarchy Verbatim]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/anti-pattern-replicating-oracles-role-hierarchy-verbatim/' | relative_url }}) | [Next: ABAC, Masking, and Cross-Engine Governance &rarr;]({{ '/03-legacy-migration-to-databricks/16-abac-masking-and-cross-engine-governance/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

