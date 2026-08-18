---
title: "Unity Catalog and the Privilege Migration"
parent: "Legacy Migration to Databricks"
nav_order: 15
has_children: true
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/
---

# Unity Catalog and the Privilege Migration

Every legacy grant you inherit -- 500 Oracle roles, a decade of one-off `GRANT SELECT ON schema.table TO user` statements, a role hierarchy nobody fully remembers building -- has to land somewhere in Unity Catalog before cutover, and "somewhere" is not "everywhere, unchanged." This section is the bridge between the reconciliation work you just finished proving semantic parity and the cost/governance sections ahead: get the privilege model wrong here and either the business gets locked out on cutover day, or -- worse -- the migration quietly over-grants and nobody notices until an audit. The goal is not a byte-for-byte copy of the Oracle role tree; it's a smaller, group-based, tag-driven model that grants the same effective access with a fraction of the objects to maintain.

```mermaid
flowchart TD
    subgraph Oracle["Legacy Oracle"]
        R1[Role: AR_CLERK]
        R2[Role: AR_MANAGER]
        R3[...500 roles]
        G1[Per-table GRANTs]
    end

    subgraph Matrix["Privilege Migration Matrix"]
        M[Object x Grantee x Privilege<br/>legacy vs target]
    end

    subgraph UC["Unity Catalog"]
        MS[Metastore] --> CAT[Catalog]
        CAT --> SCH[Schema]
        SCH --> TBL[Table / View]
        GRP[Groups] -->|USE CATALOG, USE SCHEMA, SELECT| CAT
        GRP -->|schema-level grants| SCH
        TAG[Tags: pii, domain, tier] -.governs.-> TBL
    end

    Oracle -->|inventory & translate| Matrix
    Matrix -->|collapse to ~12 tags + groups| UC


## Lectures

| # | Lecture | Est. Read |
|---|---------|-----------|
| 1 | [Unity Catalog Architecture]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/unity-catalog-architecture/' | relative_url }}) | 3 min read |
| 2 | [The Security Layer Cake]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-security-layer-cake/' | relative_url }}) | 3 min read |
| 3 | [The Privilege Migration Matrix]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-privilege-migration-matrix/' | relative_url }}) | 3 min read |
| 4 | [500 Roles to 12 Tags Without a Tagging Explosion]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/500-roles-to-12-tags-without-a-tagging-explosion/' | relative_url }}) | 3 min read |
| 5 | [Anti-Pattern: Replicating Oracle's Role Hierarchy Verbatim]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/anti-pattern-replicating-oracles-role-hierarchy-verbatim/' | relative_url }}) | 3 min read |
| 6 | [Check Your Knowledge]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/check-your-knowledge/' | relative_url }}) | Quiz |

<!-- prevnext:start -->

---

| [&larr; Previous: Check Your Knowledge]({{ '/03-legacy-migration-to-databricks/14-the-parallel-run-and-zero-downtime-cutover/check-your-knowledge/' | relative_url }}) | [Next: Unity Catalog Architecture &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/unity-catalog-architecture/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

