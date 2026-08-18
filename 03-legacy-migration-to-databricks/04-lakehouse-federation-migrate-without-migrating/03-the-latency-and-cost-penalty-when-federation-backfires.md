---
title: "The Latency and Cost Penalty: When Federation Backfires"
parent: "Lakehouse Federation: Migrate Without Migrating"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/the-latency-and-cost-penalty-when-federation-backfires/
read_minutes: 3
---

# The Latency and Cost Penalty: When Federation Backfires
{: .no_toc }

*Estimated read: 3 min*

Query pushdown makes federation feel free the first time you try it against a small table. It stops
feeling free the moment you point it at your busiest one, and the reason is worth understanding
precisely rather than treating federation as a black box that's "sometimes slow."

**What pushdown does and doesn't save you.** A filter, a `GROUP BY`, or a straightforward join
pushes down cleanly to Oracle or SQL Server and executes there, so the network only carries the
result set. But not everything pushes down -- a window function combined with a Databricks-side
function the source system doesn't have, or a join between a foreign table and a native Delta table,
often can't execute entirely on the source side. When that happens, Databricks has to pull more raw
data across the network than the query's final output implies, and that pull happens on *every single
query execution*, not once.

**Where this compounds into real cost:**

- **Latency stacks per query, not per session.** Every query against a foreign table pays the
  network round-trip and the source system's own query planning time, on top of Databricks' own
  execution time. A dashboard that fires twenty federated queries on page load pays that round-trip
  twenty times, every time anyone opens it.
- **You're still paying for the legacy system's compute.** Federation doesn't reduce load on the
  Oracle or SQL Server instance -- if anything it adds a new class of caller. A high-traffic BI tool
  federated against a legacy production system can push it into the same contention issues (`enq:
  TX - row lock contention`, CPU saturation) you were trying to escape by migrating in the first
  place, competing directly with the OLTP or batch workloads still running there.
- **Full-table scans without effective pushdown are the worst case.** A federated table with no
  selective filter, queried by an analyst doing exploratory `SELECT *` work, pulls the entire table
  across the network on every run -- for a multi-terabyte table, that's minutes of latency and real
  egress cost, repeated for every analyst who runs it.

The heat map from the autopsy section is exactly the signal you already have to predict this before
it happens: a table with **high access frequency and high data volume** is precisely the kind of
table federation handles worst, because every one of those frequent accesses re-pays the round-trip
cost.

{: .important }
> Use the workload inventory's `access_frequency_30d` and `storage_gb` columns as your federation
> triage: low-frequency, low-volume tables are excellent, low-risk federation candidates. High-
> frequency, high-volume tables -- your busiest reporting tables, your most-queried fact tables --
> should be prioritized for actual physical migration precisely *because* federation performs worst
> exactly where you'd be tempted to rely on it most.

None of this makes federation a bad tool -- it makes it the wrong tool for hot tables specifically.
The next lecture shows how to sequence federation and physical migration together deliberately, so
you get federation's early-access benefits without leaving your hottest tables federated indefinitely.

<!-- prevnext:start -->

---

| [&larr; Previous: Configuring a Federated Connection to Oracle and SQL Server]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/configuring-a-federated-connection-to-oracle-and-sql-server/' | relative_url }}) | [Next: Federate-then-Migrate: The Phased Pattern &rarr;]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/federate-then-migrate-the-phased-pattern/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

