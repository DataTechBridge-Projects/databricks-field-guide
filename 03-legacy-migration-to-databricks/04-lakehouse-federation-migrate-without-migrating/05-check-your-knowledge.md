---
title: "Check Your Knowledge"
parent: "Lakehouse Federation: Migrate Without Migrating"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you've learned from this section before moving on to schema translation with Lakebridge.

1. What are the two Unity Catalog objects required to set up Lakehouse Federation to an external database?
   A. A cluster and a warehouse
   B. A connection and a foreign catalog
   C. A metastore and a workspace
   D. A pipeline and a job

2. What does "query pushdown" mean in the context of Lakehouse Federation?
   A. Data is copied nightly from the source system into Delta tables
   B. As much of the query as the source system supports is translated and executed on the source system, returning only the result set
   C. Databricks always pulls the entire table before filtering
   D. Queries are cached indefinitely in Unity Catalog

3. Which privilege is required on the Unity Catalog metastore before creating a connection?
   A. `CREATE CONNECTION`
   B. `MODIFY CLUSTER`
   C. `CREATE VOLUME`
   D. `MANAGE ACCOUNT`

4. Per this section's guidance, how should credentials be supplied in a `CREATE CONNECTION` statement?
   A. As plaintext strings directly in the SQL statement
   B. Referenced from a Databricks secret scope rather than plaintext
   C. Embedded in the foreign catalog name
   D. Credentials are not required for federated connections

5. Why does a federated dashboard that fires twenty queries on page load pay a latency cost twenty times over?
   A. Because Unity Catalog throttles federated queries after the first one
   B. Because each query against a foreign table pays its own network round-trip and source-side query planning time, independently
   C. Because foreign catalogs cache only one query result at a time
   D. Because SQL Server and Oracle share a single connection pool by default

6. According to the workload-inventory-based triage in this section, which tables are the worst fit for staying federated long-term?
   A. Low access frequency, low data volume tables
   B. Tables that are never queried
   C. High access frequency, high data volume tables
   D. Tables with fewer than 10 columns

7. What happens to load on the legacy source system when it is federated for BI/dashboard use?
   A. Load decreases because Databricks caches all results
   B. The legacy system's compute is still consumed by every federated query, potentially competing with existing OLTP/batch workloads
   C. The legacy system is automatically taken offline
   D. Federation eliminates the need for the legacy system's compute entirely

8. In the federate-then-migrate phased pattern, what is stage 1?
   A. Decommission the source system
   B. Federate the entire estate first, before any table physically migrates
   C. Migrate only the highest-volume tables first
   D. Delete the foreign catalog

9. In the phased pattern's stage 3 example, what technique lets downstream consumers stay agnostic to whether a given table has migrated yet?
   A. A stable view layer that points at either the foreign table or the migrated Delta table underneath
   B. Renaming every foreign table to match its future Delta table name
   C. Duplicating every dashboard for federated vs. migrated data
   D. Disabling Unity Catalog audit logging during migration

10. What does this section identify as the sign that the federate-then-migrate pattern has stalled rather than completed?
    A. A foreign catalog that was dropped immediately after creation
    B. A federated connection still live long after migration was declared complete
    C. A workload inventory with no federated tables at all
    D. A view layer pointing entirely at Delta tables

## Answer Key

1. **B** -- A connection (credentials/network details) and a foreign catalog (mirrors the external schema) are the two required objects.
2. **B** -- Pushdown translates and executes as much of the query as possible on the source system, returning only the result set across the network.
3. **A** -- `CREATE CONNECTION` privilege on the metastore is required, alongside `CREATE CATALOG` for the foreign catalog step.
4. **B** -- Credentials should be referenced via `secret('scope', 'key')` from a Databricks secret scope, never plaintext in the statement.
5. **B** -- Each federated query independently pays its own network round-trip and source-side planning cost; this cost is per-query, not per-session.
6. **C** -- High access frequency combined with high data volume is exactly where federation's repeated round-trip cost compounds worst.
7. **B** -- Federation adds a new class of caller to the legacy system; it does not reduce or eliminate its compute load.
8. **B** -- Stage 1 is federating the entire estate first, before any physical migration, to give a single governed surface immediately.
9. **A** -- A stable view whose target swaps from the foreign table to the migrated Delta table lets consumers stay unaware of migration progress.
10. **B** -- A federated connection still live long after "migration complete" signals the phased pattern stalled partway through rather than finishing.

<!-- prevnext:start -->

---

| [&larr; Previous: Federate-then-Migrate: The Phased Pattern]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/federate-then-migrate-the-phased-pattern/' | relative_url }}) | [Next: Schema Translation with Lakebridge (and Its Blind Spots) &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

