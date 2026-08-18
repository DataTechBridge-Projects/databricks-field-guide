---
title: "Configuring a Federated Connection to Oracle and SQL Server"
parent: "Lakehouse Federation: Migrate Without Migrating"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/configuring-a-federated-connection-to-oracle-and-sql-server/
read_minutes: 3
---

# Configuring a Federated Connection to Oracle and SQL Server
{: .no_toc }

*Estimated read: 3 min*

Setting up a federated connection is two SQL statements, but the prerequisites around them are
where most first attempts stall. Before you write a `CREATE CONNECTION`, confirm: Unity Catalog is
enabled on the workspace, the compute you'll query through is on Databricks Runtime 16.1+ (or a
Pro/Serverless SQL warehouse), and you hold the `CREATE CONNECTION` privilege on the metastore plus
`CREATE CATALOG` for the foreign catalog step. Network connectivity from that compute to the source
database -- VPC peering, a private endpoint, or a firewall rule opening the right port -- has to
already exist; federation doesn't create network paths, it uses ones you've already built.

**Connecting to [Oracle](https://docs.databricks.com/aws/en/query-federation/oracle):**

```sql
CREATE CONNECTION legacy_oracle_conn TYPE oracle
OPTIONS (
  host 'oracle-prod.internal.example.com',
  port '1521',
  user secret('migration_secrets', 'oracle_user'),
  password secret('migration_secrets', 'oracle_password'),
  encryption_protocol 'Transport Layer Security'
);

CREATE FOREIGN CATALOG legacy_oracle
USING CONNECTION legacy_oracle_conn
OPTIONS (database 'ORCLPDB1');
```

Oracle requires Server-side Native Network Encryption at a minimum `ACCEPTED` level, or TLS if you're
connecting to an Oracle Cloud instance -- confirm which your DBA has configured before you pick
`encryption_protocol`.

**Connecting to [SQL Server](https://docs.databricks.com/aws/en/query-federation/sql-server):**

```sql
CREATE CONNECTION legacy_sqlserver_conn TYPE sqlserver
OPTIONS (
  host 'sqlserver-prod.internal.example.com',
  port '1433',
  user secret('migration_secrets', 'sqlserver_user'),
  password secret('migration_secrets', 'sqlserver_password')
);

CREATE FOREIGN CATALOG legacy_sqlserver
USING CONNECTION legacy_sqlserver_conn
OPTIONS (database 'LegacyEDW');
```

{: .important }
> Use `secret('scope', 'key')` referencing a Databricks secret scope for `user` and `password` in
> every real connection -- never plaintext credentials in the `CREATE CONNECTION` statement itself.
> Databricks documentation explicitly recommends this, and it's the same discipline you already use
> for any service principal credential elsewhere in Unity Catalog.

Once the foreign catalog exists, it behaves like any other catalog in the metastore: it shows up in
Catalog Explorer, you grant `USE CATALOG` / `SELECT` on it exactly like a native one, and every query
against it is captured in Unity Catalog's audit log and `system.access.audit` alongside your Delta
table queries. A stored-procedure-heavy legacy schema will surface every table and view in that
schema as a foreign table automatically -- you don't need to enumerate them individually.

```sql
-- Confirm the foreign catalog is browsable
SHOW SCHEMAS IN legacy_oracle;

-- Query a foreign table exactly like a native one
SELECT customer_id, order_total
FROM legacy_oracle.sales.orders
WHERE order_date >= DATE '2026-01-01';
```

That last query looks identical to querying a Delta table, and that's the point -- but under the
hood, the `WHERE` clause got pushed down and executed inside Oracle, with only the filtered rows
crossing the network. What that pushdown does and doesn't save you is exactly where the next lecture
picks up.

<!-- prevnext:start -->

---

| [&larr; Previous: What Lakehouse Federation Actually Does]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/what-lakehouse-federation-actually-does/' | relative_url }}) | [Next: The Latency and Cost Penalty: When Federation Backfires &rarr;]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/the-latency-and-cost-penalty-when-federation-backfires/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

