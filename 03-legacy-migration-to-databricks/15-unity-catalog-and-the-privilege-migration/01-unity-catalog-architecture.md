---
title: "Unity Catalog Architecture"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/unity-catalog-architecture/
read_minutes: 3
---

# Unity Catalog Architecture
{: .no_toc }

*Estimated read: 3 min*

If you spent years administering Oracle, you already have a mental model for a namespace: `schema.table`, with schemas owned by a user and privileges granted per-object or per-role. **Unity Catalog** replaces that two-level namespace with a three-level one -- `catalog.schema.table` -- and, more importantly, moves governance out of each individual warehouse and into a single control plane that spans every workspace attached to it.

The hierarchy, top to bottom:

- **Metastore** -- the top-level container for a region. One metastore is attached to one or more workspaces; it holds the catalogs, the shared audit log, and the identity federation (users, groups, service principals) that every attached workspace inherits. This is the closest thing to "the whole Oracle instance," except it isn't tied to a single compute engine -- a Databricks SQL warehouse, an all-purpose cluster, and a serverless job in three different workspaces all resolve grants against the same metastore.
- **Catalog** -- the first namespace level, typically mapped to an environment or a business domain (`dev`, `uat`, `prod`, or `finance`, `sales`). This is roughly where an Oracle "database" or a SQL Server "database" sits conceptually, though Unity Catalog catalogs are logical containers, not separate storage engines.
- **Schema** -- the second namespace level, the direct analog of an Oracle schema: a logical grouping of tables, views, volumes, and functions that usually corresponds to a subject area (`accounts_receivable`, `general_ledger`).
- **Table / View / Volume / Function / Model** -- the objects themselves. A **volume** is new vocabulary for legacy DBAs: it's a governed pointer to non-tabular files (PDFs, images, staged CSVs) living in cloud storage, playing the role your Oracle directory object or a shared network drive used to play, but with the same grant model as a table.

Underneath the logical hierarchy sits the physical connection to cloud storage: a **storage credential** (the IAM role or service principal Unity Catalog assumes) paired with an **external location** (the S3 path that credential is scoped to). Every managed table in a catalog inherits its storage location from the catalog or schema's default location unless a table is created as `EXTERNAL` and points at its own external location explicitly -- the Unity Catalog equivalent of the Oracle DBA granting a directory object and telling a schema owner "your tablespace lives here."

{: .important }
> A metastore is provisioned once per region per Databricks account and attached to every workspace in that region. Migrating privileges is therefore not a per-workspace exercise -- get the metastore-level catalog and schema design right once, and every workspace that attaches to it inherits the same governance for free.

The practical implication for a migration: before translating a single `GRANT` statement, decide the catalog/schema layout first. Most legacy shops that mapped one Oracle instance per environment naturally collapse to one catalog per environment (`dev`, `uat`, `prod`) with schemas mirroring the old Oracle schema names -- a mapping simple enough that the privilege matrix in the next lecture can be built mechanically rather than redesigned from scratch. See the [Unity Catalog overview](https://docs.databricks.com/aws/en/data-governance/unity-catalog/) for the full object model.

<!-- prevnext:start -->

---

| [&larr; Previous: Unity Catalog and the Privilege Migration]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/' | relative_url }}) | [Next: The Security Layer Cake &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-security-layer-cake/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

