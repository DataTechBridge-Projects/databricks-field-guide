---
title: "Databricks Compute Cluster"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 7
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-compute-cluster/
read_minutes: 19
---

# Databricks Compute Cluster
{: .no_toc }

*Estimated read: 19 min*

Every notebook cell, every SQL query, every scheduled job runs *somewhere* -- that somewhere is a
**cluster**: a set of virtual machines running Spark, managed by Databricks. This lecture is the
deep dive on cluster types, sizing, and the configuration choices that determine both performance
and cost, closing out this section before Delta Lake in Section 5.

## Two clusters, functionally

Databricks distinguishes clusters primarily by *purpose*, which determines lifecycle and billing
behavior:

- **All-purpose clusters** -- created interactively, shared by multiple users and notebooks,
  running until explicitly terminated or auto-terminated after idle time. This is what you attach
  a notebook to while developing.
- **Jobs clusters** -- created automatically for a single scheduled job run, then torn down
  immediately when the job finishes. No idle cost, because there's no idle period -- the cluster's
  entire lifetime is the job's runtime.

**Key term:** this distinction matters directly for cost. An all-purpose cluster left running
between sessions bills continuously; a jobs cluster only ever exists for the duration of the work
it was created for. [Part 3's FinOps content]({{ '/03-legacy-migration-to-databricks/17-the-cost-iceberg-and-compute-arbitrage/' | relative_url }})
covers this tradeoff in real financial terms once you're managing production spend, not just a
learning account.
{: .important }

## Serverless vs. classic, revisited for compute specifically

The earlier lecture on platform access introduced serverless vs. classic at the *workspace* level;
the same distinction applies at the *cluster* level:

| | Classic cluster | Serverless compute |
|---|---|---|
| Provisioning | You choose instance type, node count, autoscaling range | Fully managed, no instance selection |
| Startup time | Roughly 1-5 minutes (VM boot) | Seconds |
| Networking | Runs in your VPC (customer-managed) or a Databricks-provisioned one | Runs in Databricks' account |
| Billing | DBUs + underlying AWS EC2/EBS cost | Single unified DBU rate |
| Instance control | Full control (GPU types, memory-optimized, spot instances) | None |

## Configuring a classic cluster

The cluster creation form covers several decisions worth understanding, not just clicking through:

**Databricks Runtime version.** A curated, versioned bundle of Spark plus Databricks-specific
optimizations (including Photon) and pre-installed libraries. Newer runtimes generally mean better
performance and current security patches -- pin to a specific version for production jobs so a
runtime upgrade can't silently change behavior mid-project, but stay reasonably current rather than
freezing indefinitely on an old version.

**Node type and count.** The instance type for driver and worker nodes (e.g. memory-optimized vs.
compute-optimized AWS instance families), and how many workers. More workers means more parallel
processing capacity -- the direct analogue of adding more nodes to a distributed warehouse cluster.

**Autoscaling.** Set a minimum and maximum worker count, and Databricks adds or removes workers
based on actual load. For workloads with unpredictable or bursty volume (a common shape for
ingestion pipelines pulling from source systems with irregular file arrival), this avoids
permanently provisioning for peak load.

```text
Example autoscaling range: min 2 workers, max 8 workers
- Light query load  -> scales down toward 2
- Heavy nightly batch -> scales up toward 8
- Billed only for workers actually running at any moment
```

**Spot instances.** AWS spot instances offer significant cost savings (often 60-90% off on-demand
pricing) in exchange for the possibility AWS reclaims the instance with short notice. Databricks
handles this gracefully for worker nodes -- a reclaimed spot worker is replaced automatically, and
Spark's fault tolerance re-processes any lost work. Using spot instances for the **driver** node is
generally discouraged, since losing the driver fails the entire job, not just one worker's
partition of the work.

**Auto-termination.** An idle timeout (e.g. 30 or 60 minutes with no active commands) after which
Databricks terminates the cluster automatically. This is the single most important setting for
controlling accidental cost on an interactive, all-purpose cluster -- set it low enough that a
forgotten notebook doesn't run overnight.
{: .important }

## Cluster policies

A **cluster policy** is an admin-defined template that constrains what configuration options a
user can actually choose -- e.g. locking node type to a specific instance family, capping maximum
worker count, or forcing spot-instance usage for a given team. This is the mechanism a real
organization uses to prevent an individual engineer from accidentally spinning up an expensive
GPU cluster for a routine ETL job, without needing to manually review every cluster creation.
Policies get direct hands-on treatment in
[Part 3's cost-management content]({{ '/03-legacy-migration-to-databricks/17-the-cost-iceberg-and-compute-arbitrage/' | relative_url }}),
where enforcing jobs-compute-only policies is one of the concrete cost levers covered.

## Instance pools

An **instance pool** keeps a set of idle-but-ready VM instances on standby, so a cluster attaching
to the pool starts almost instantly instead of waiting for fresh VM provisioning -- at the cost of
paying for that idle standby capacity. Worth it for workloads where startup latency directly
impacts users (e.g. an interactive BI tool's query cluster); rarely worth it for overnight batch
jobs where a few extra minutes of startup time is irrelevant.

## SQL warehouses: a third compute type

Distinct from all-purpose and jobs clusters, a **SQL warehouse** is compute optimized specifically
for SQL/BI workloads -- what you'd attach a dashboard or a BI tool's live query connection to,
rather than a general-purpose notebook. Available in both serverless and classic variants. If your
prior role involved tuning a warehouse specifically for concurrent BI query load, a SQL warehouse
is the closest direct equivalent inside Databricks.

## Mapping this onto what you already know

| Legacy warehouse concept | Databricks equivalent |
|---|---|
| A dedicated ETL processing node/instance | Jobs cluster (spun up per run, torn down after) |
| Shared interactive query environment | All-purpose cluster |
| Fixed hardware, sized for peak load | Autoscaling classic cluster, or serverless |
| DBA-enforced resource limits per team | Cluster policies |
| BI-optimized query engine | SQL warehouse |

None of this changes the fundamental sizing question you already know how to reason about --
how much data, how much parallelism, how much concurrency -- it just changes where those knobs
live and how granularly you can scope them.

For the full, current official reference on every compute type and configuration option beyond
what this lecture covers, see
[Databricks compute documentation](https://docs.databricks.com/aws/en/compute/).

This closes out the hands-on tour of the workspace. The next lecture is a short knowledge check
before Section 5 moves into Delta Lake -- the storage layer underneath everything you've configured
compute to process.

<!-- prevnext:start -->

---

| [&larr; Previous: Workspace Files vs Git Folders]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/workspace-files-vs-git-folders/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
