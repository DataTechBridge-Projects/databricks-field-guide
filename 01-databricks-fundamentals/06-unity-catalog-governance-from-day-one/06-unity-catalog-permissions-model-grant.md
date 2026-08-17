---
title: "Unity Catalog Permissions Model - GRANT"
parent: "Unity Catalog - Governance from day one"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/unity-catalog-permissions-model-grant/
read_minutes: 15
---

# Unity Catalog Permissions Model - GRANT
{: .no_toc }

*Estimated read: 15 min*

The last two lectures used `GRANT` statements without fully unpacking them. This lecture is the
complete permissions model -- privilege types, inheritance, ownership, and who's allowed to grant
what to whom.

## The core privileges

```sql
GRANT USE CATALOG ON CATALOG sales TO `data-engineering`;
GRANT USE SCHEMA ON SCHEMA sales.orders TO `data-engineering`;
GRANT CREATE SCHEMA ON CATALOG sales TO `data-engineering`;
GRANT CREATE TABLE ON SCHEMA sales.orders TO `data-engineering`;
GRANT SELECT ON TABLE sales.orders.fact_orders TO `data-engineering`;
GRANT MODIFY ON TABLE sales.orders.fact_orders TO `data-engineering`;
```

| Privilege | Grants the ability to |
|---|---|
| `USE CATALOG` | See and navigate into a catalog at all -- the baseline "can this identity even know this exists" |
| `USE SCHEMA` | See and navigate into a schema |
| `CREATE SCHEMA` | Create new schemas within a catalog |
| `CREATE TABLE` | Create new tables within a schema |
| `SELECT` | Read data from a table/view |
| `MODIFY` | Insert, update, delete, merge into a table |

**Key term:** `USE CATALOG` and `USE SCHEMA` are **prerequisites**, not standalone useful grants --
`SELECT` on a table without `USE CATALOG`/`USE SCHEMA` on its parents does nothing, because the
identity can't navigate to the table at all. This trips people up constantly: granting `SELECT` on
a table and forgetting the parent `USE` grants results in "access denied" that looks like the
`SELECT` grant didn't work, when actually it's the missing parent grant.
{: .important }

## Privilege inheritance

Grants at a higher level (catalog) flow down to everything beneath it (schemas, tables) unless
more specific grants exist. Granting `SELECT` at the **catalog** level gives read access to every
current and *future* table in every schema within it -- useful for broad, stable access patterns;
dangerous if you actually meant to scope access to one schema and granted one level too high by
mistake.

```sql
-- Grants SELECT on every table in every schema of sales, present and future
GRANT SELECT ON CATALOG sales TO `analysts`;

-- Grants SELECT only within the orders schema
GRANT SELECT ON SCHEMA sales.orders TO `analysts`;
```

## Ownership

Every securable object has an **owner** -- by default, whoever created it -- who holds all
privileges on it implicitly and can grant privileges to others without needing an explicit
`MANAGE` grant. Ownership can be transferred:

```sql
ALTER TABLE sales.orders.fact_orders OWNER TO `data-engineering`;
```

Transferring ownership to a **group** rather than leaving it with the individual who happened to
create the table is worth doing deliberately for anything production-bound -- otherwise "who owns
this table" quietly becomes "whoever created it two years ago and may no longer work here."

## Who can grant what

Four categories of identity can grant privileges on an object: the object's **owner**, an owner of
a parent (catalog/schema owner can grant on objects within it), anyone holding `MANAGE` privilege
on the object, and **metastore admins** (who have implicit administrative reach across the entire
metastore). Account admins and workspace admins also hold relevant default privileges depending on
scope.

## Viewing and revoking

```sql
SHOW GRANTS ON TABLE sales.orders.fact_orders;
REVOKE SELECT ON TABLE sales.orders.fact_orders FROM `contractors`;
```

`REVOKE` is the direct inverse of `GRANT` -- removes exactly the privilege specified, leaving any
other grants on the object untouched.

## Default access on workspace catalogs

Worth knowing explicitly: workspace users automatically receive `USE CATALOG` and schema-creation
privileges on a workspace's **default catalog** (the one auto-provisioned per workspace) --
meaning a fresh workspace isn't a completely locked-down blank slate by default. For anything
beyond casual exploration, the deliberate multi-catalog structure from earlier lectures in this
section is what actually governs access, not the default catalog's baseline permissiveness.
{: .important }

For the complete official reference on every privilege type and the full grant-authority matrix,
see [Manage privileges in Unity Catalog](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/).

<!-- prevnext:start -->

---

| [&larr; Previous: Implementing Users, Groups, and Access Control]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-users-groups-and-access-control/' | relative_url }}) | [Next: Implementing Unity Catalog Permission for Catalog and Schema &rarr;]({{ '/01-databricks-fundamentals/06-unity-catalog-governance-from-day-one/implementing-unity-catalog-permission-for-catalog-and-schema/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
