---
title: "Unity Catalog Permissions for Tables and Volumns"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-for-tables-and-volumns/
read_minutes: 11
---

# Unity Catalog Permissions for Tables and Volumns
{: .no_toc }

*Estimated read: 11 min*

This section closes with the finest-grained permission level: individual tables and volumes,
including row- and column-level restrictions -- the tools you'll reach for when "everyone on the
team can read this schema" isn't precise enough.

## Table-level grants, precisely

```sql
GRANT SELECT ON TABLE sales.orders.fact_orders TO `analysts`;
GRANT SELECT, MODIFY ON TABLE sales.orders.fact_orders TO `data-engineering`;
GRANT SELECT ON VIEW sales.orders.customer_summary_view TO `marketing-team`;
```

Table-level (and view-level) grants override schema-level defaults for that specific object --
useful when one table in a schema is more sensitive than the rest, and schema-level `SELECT`
alone would over-expose it.

## Volume permissions

```sql
GRANT READ VOLUME ON VOLUME sales.landing.raw_files TO `data-engineering`;
GRANT WRITE VOLUME ON VOLUME sales.landing.raw_files TO `ingestion-service-principal`;
```

`READ VOLUME` and `WRITE VOLUME` are the file-level equivalent of `SELECT`/`MODIFY` on a table --
governing who can read from or write to a volume's underlying files, without needing separate S3-
level IAM grants for individual users. This is what makes a landing-zone volume safely shareable:
an ingestion job's service principal gets write access, engineers get read access for debugging,
and nobody gets direct S3 access outside Unity Catalog at all.

## Beyond simple grants: row filters and column masks

For genuinely sensitive columns -- PII, financial detail -- table-level `SELECT`/deny is often too
coarse. Unity Catalog supports **row filters** (restricting which rows a query returns based on
the querying identity) and **column masks** (redacting or transforming specific column values)
applied as functions at the table level:

```sql
CREATE FUNCTION mask_ssn(ssn STRING)
RETURNS STRING
RETURN CASE
  WHEN is_member('hr-team') THEN ssn
  ELSE 'XXX-XX-' || right(ssn, 4)
END;

ALTER TABLE sales.employees.records
ALTER COLUMN ssn SET MASK mask_ssn;
```

**Key term:** this is a preview of the **ABAC** (attribute-based access control) pattern
[Part 3 covers in full]({{ '/03-legacy-migration-to-databricks/16-abac-masking-and-cross-engine-governance/' | relative_url }})
for a real migration -- the same principle (masking driven by tags/group membership rather than
maintaining separate masked views by hand) at a larger, governed scale.
{: .important }

## A checklist for a new table's access setup

Before considering a new production table's permissions "done," confirm:

1. Parent catalog and schema have appropriate `USE CATALOG`/`USE SCHEMA` grants for every group
   that needs table-level access.
2. Table-level `SELECT`/`MODIFY` scoped to exactly who needs it -- not inherited broad
   catalog/schema grants standing in for a decision you haven't actually made.
3. Sensitive columns identified and masked, or the table's access scoped narrowly enough that
   masking isn't needed.
4. Ownership set to a group, not an individual.
5. Both an allowed and a denied case tested, per the previous lecture's practice.

This closes out Unity Catalog. The section's knowledge check is next, before Section 7 moves into
Medallion Architecture -- where governed catalogs, schemas, and tables become the concrete
bronze/silver/gold structure every pipeline in this guide builds toward.

<!-- prevnext:start -->

---

| [&larr; Previous: Implementing Unity Catalog Permission for Catalog and Schema]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-unity-catalog-permission-for-catalog-and-schema/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
