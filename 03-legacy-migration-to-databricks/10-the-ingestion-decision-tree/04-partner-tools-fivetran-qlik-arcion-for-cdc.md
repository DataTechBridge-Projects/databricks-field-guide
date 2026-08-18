---
title: "Partner Tools: Fivetran, Qlik, Arcion for CDC"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/partner-tools-fivetran-qlik-arcion-for-cdc/
read_minutes: 2
---

# Partner Tools: Fivetran, Qlik, Arcion for CDC
{: .no_toc }

*Estimated read: 2 min*

Neither JDBC bulk nor a hand-rolled Auto Loader export pipeline is a good fit for one specific
requirement: continuous, row-level **change data capture (CDC)** off a live transactional
database, with sub-minute latency and no sustained batch load hitting the source. That's the gap
partner CDC tools exist to fill, and it's worth knowing when reaching for one beats building the
equivalent yourself.

## What these tools actually do

CDC tools like **Fivetran**, **Qlik Replicate**, and **Arcion** read a source database's
**transaction log** (Oracle redo logs, SQL Server's CDC/CT feature, MySQL's binlog) rather than
querying tables directly. That's the same mechanism a legacy Oracle GoldenGate or Attunity
replication setup used -- if your organization already runs one of those, a migration project is
often just repointing the existing CDC pipeline at a Databricks target instead of a legacy
warehouse. The advantages over a query-based pull:

- **No repeated table scans.** Reading the log captures every insert/update/delete as it happens,
  instead of re-querying the full table (or even an incremental slice of it) on a schedule.
- **Minimal source-system load.** Log-based CDC has a far lighter footprint on the production OLTP
  database than repeated JDBC queries, which is often the deciding factor when a DBA is reluctant
  to grant broader query access.
- **Row-level granularity with ordering.** Every change arrives as a discrete, ordered event --
  exactly what an SCD Type 2 history table or a `MERGE INTO` upsert needs as input.

Databricks documents this landscape as part of [Lakeflow
Connect](https://docs.databricks.com/aws/en/ingestion/lakeflow-connect/), which distinguishes
managed database (CDC) connectors, alongside SaaS, file, and query-based connector types --
Lakeflow Connect's own managed database connectors cover some of the same sources these
third-party tools target, and are worth evaluating first when the source is one they support
natively.

## Where each tool tends to fit

| Tool | Typical strength | Where it shows up in a migration |
|---|---|---|
| **Fivetran** | Broad catalog of managed SaaS + database connectors, low operational overhead | SaaS sources (Salesforce, NetSuite) and databases where a fully managed, low-maintenance connector matters more than fine-grained control |
| **Qlik Replicate** | Mature, enterprise-grade log-based replication, wide legacy database support (Oracle, DB2, mainframe) | Large enterprise migrations replacing an existing GoldenGate/Attunity-style CDC setup |
| **Arcion** | High-throughput, distributed CDC engineered for zero data loss at scale | High-volume OLTP sources where sustained throughput and exactly-once delivery under load are the primary requirement |

## Build vs. buy

The honest trade-off: building a comparable log-based CDC pipeline yourself is a substantial
undertaking -- log parsing, exactly-once delivery, schema drift handling, and failure recovery are
each nontrivial engineering problems that these vendors have already solved. The cases where a
migration team still builds it themselves tend to be narrow: a source system with no supported
connector, a strict no-third-party-data-access security policy, or volume low enough that a
scheduled JDBC/Auto Loader pattern is genuinely sufficient and a CDC tool's licensing cost isn't
justified.

{: .important }
> A partner CDC tool solves *ingestion* into a bronze Delta table -- it does not replace the
> Lakeflow Declarative Pipeline logic (`create_auto_cdc_flow`) needed to apply those row-level
> changes into an SCD Type 2 silver table. Budget for both stages when comparing a CDC tool's cost
> against a "just use JDBC" estimate.

With batch, streaming, JDBC, Auto Loader, and CDC tools all now on the table, the next lecture
collapses every one of these choices into a single matrix you can apply directly against your
migration's workload inventory.

<!-- prevnext:start -->

---

| [&larr; Previous: JDBC Bulk vs Auto Loader: The 10TB Table Decision]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/jdbc-bulk-vs-auto-loader-the-10tb-table-decision/' | relative_url }}) | [Next: The Ingestion Pattern Matrix &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/the-ingestion-pattern-matrix/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

