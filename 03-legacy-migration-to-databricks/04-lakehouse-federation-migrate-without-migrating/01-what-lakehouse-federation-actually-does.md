---
title: "What Lakehouse Federation Actually Does"
parent: "Lakehouse Federation: Migrate Without Migrating"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/what-lakehouse-federation-actually-does/
read_minutes: 3
---

# What Lakehouse Federation Actually Does
{: .no_toc }

*Estimated read: 3 min*

Every migration section so far has assumed the eventual outcome is data physically landing in Delta
tables. **[Lakehouse Federation](https://docs.databricks.com/aws/en/query-federation/)** is the tool
for the phase before that: governed, read-only access to your legacy Oracle or SQL Server data
straight from Databricks, through Unity Catalog, with automatic query pushdown -- without copying a
single row.

Two Unity Catalog objects make this work:

- A **connection** stores the credentials and network details for reaching the external system --
  host, port, authentication, encryption settings. You create it once per source system.
- A **foreign catalog** is what mirrors that external database's schemas and tables *inside* Unity
  Catalog, so they show up in the catalog browser, can be queried with ordinary SQL, and are subject
  to the same grants and audit logging as any native Delta table -- even though the data never
  leaves the source system.

When you run a query against a foreign table, Databricks doesn't pull the entire table across the
wire and filter locally. It performs **query pushdown**: as much of the query as the source system
supports -- filters, aggregations, joins in some cases -- gets translated into that system's own SQL
dialect and executed *there*, with only the result set returned to Databricks. This is the same
principle as pushdown predicates you already know from partition pruning, applied across a network
boundary to a foreign engine instead of within Delta's own transaction log.

Why this matters for a migration specifically: it decouples "can my Databricks-side reports and
pipelines see this data" from "has this data been physically migrated yet." That decoupling is what
makes federation genuinely useful as a *migration* tool, not just a permanent integration pattern:

- **Early validation.** Point a new Databricks dashboard at a federated Oracle table before you've
  migrated anything, and confirm your downstream BI layer, semantic model, and governance policies
  all work correctly against real data -- weeks before the data itself moves.
- **Sequencing flexibility.** Migrate your highest-priority workloads first (per the 3-R decision)
  while lower-priority tables stay federated, queryable, and fully governed under Unity Catalog the
  entire time -- instead of blocking every downstream consumer until the *entire* estate has moved.
- **A safety valve during cutover.** If a migrated table's Databricks-side numbers don't reconcile
  yet, a federated connection back to the legacy source gives you a governed fallback path rather
  than an unofficial VPN tunnel someone opens under pressure.

{: .important }
> Federation is not a replacement for migration -- it's a bridge. Every foreign table still incurs a
> live round-trip to the source system on every query, which means the legacy platform you're trying
> to retire is still running, still licensed, and still in your critical path for as long as
> anything stays federated. The next lecture gets hands-on with configuring a connection; the one
> after that covers exactly where this bridge starts to cost more than it saves.

<!-- prevnext:start -->

---

| [&larr; Previous: Lakehouse Federation: Migrate Without Migrating]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/' | relative_url }}) | [Next: Configuring a Federated Connection to Oracle and SQL Server &rarr;]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/configuring-a-federated-connection-to-oracle-and-sql-server/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

