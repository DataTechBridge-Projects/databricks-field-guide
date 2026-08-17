---
title: "Apache Spark to Data Engineering Platform"
parent: "Introduction"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/02-introduction/apache-spark-to-data-engineering-platform/
read_minutes: 8
---

# Apache Spark to Data Engineering Platform
{: .no_toc }

*Estimated read: 8 min*

Databricks was founded by the original creators of **Apache Spark**, and it's easy to conflate the
two -- but they're not the same thing, and understanding the difference clarifies a lot of what
you'll see in the rest of this guide. Spark is the open-source processing engine. Databricks is a
managed platform that wraps Spark with everything a production data team actually needs around it:
governance, orchestration, storage management, and a collaborative workspace.

## What Spark actually is

**Apache Spark** is a distributed processing engine for large-scale data. Its core abstraction is
the **DataFrame** -- a table-like structure, distributed across a cluster of machines, that you
manipulate with transformations (`filter`, `select`, `groupBy`, `join`) instead of hand-written
loops. If you've used a SQL engine that parallelizes a query across nodes, the mental model is
similar; Spark just does it for general-purpose transformations, not only SQL, and across data
too large to fit on one machine.

```python
# A transformation any SQL-background engineer can read almost immediately
orders_df = (
    spark.table("bronze.orders")
    .filter("order_status != 'cancelled'")
    .groupBy("customer_id")
    .agg({"order_total": "sum"})
)
```

Two things matter about how Spark executes this:

- **Lazy evaluation.** Nothing in the snippet above actually runs until you call an action like
  `.show()` or `.write()`. Spark builds a query plan first, the same way a warehouse's optimizer
  builds a plan before executing a SQL statement -- and for the same reason: it can rewrite and
  optimize the whole chain before touching any data.
- **In-memory, distributed execution.** Where your legacy ETL tool likely staged intermediate
  results to disk between steps (or relied on the warehouse's own temp tables), Spark keeps data
  in memory across the cluster where possible, which is most of why Spark jobs on equivalent data
  volumes run faster than a warehouse-scripted equivalent.

## What Spark alone doesn't give you

Running open-source Spark yourself means you're also responsible for: provisioning and patching
the cluster, managing who can access what data, versioning and scheduling your jobs, monitoring
failures, and building any kind of governed table catalog by hand. None of that is part of Spark
itself -- it's infrastructure you'd bolt on, the same way you once bolted a scheduler and an access
control layer onto a bare warehouse engine.

## What Databricks adds on top

**Databricks is that infrastructure, managed.** Specifically, on top of open-source Spark it adds:

| Layer | What it gives you |
|---|---|
| **Photon** | A native, vectorized query engine that accelerates SQL and DataFrame operations, largely transparent to your code |
| **Delta Lake** | ACID transactions, schema enforcement, and time travel on top of plain files |
| **Unity Catalog** | A single, centrally governed table/permissions catalog across every workspace |
| **Notebooks & workspace** | A collaborative environment to write, run, and share Spark code |
| **Cluster management** | Autoscaling, spot-instance handling, and job-scoped compute you don't hand-provision |

**Key term:** **Photon** is Databricks' C++-based execution engine that replaces Spark's default
JVM execution for supported operations -- you don't opt into it explicitly for most workloads, it
just makes qualifying queries faster.

## Why this distinction matters for you

When you read Databricks documentation or job logs, you'll see both vocabularies mixed constantly:
Spark's terms (DataFrame, executor, shuffle, stage) for what's actually running, and Databricks'
terms (cluster, workspace, Unity Catalog, Lakeflow) for how it's managed and governed. Knowing
which layer a given term belongs to will save you time later when you're debugging a slow job and
need to know whether the fix is a Spark-level tuning change or a Databricks platform setting.

The official [Apache Spark documentation](https://spark.apache.org/docs/latest/) is worth
bookmarking directly -- its SQL/DataFrame programming guide is the canonical reference for the
transformation API you'll use in every PySpark example in this guide.

<!-- prevnext:start -->

---

| [&larr; Previous: Introduction to Data Engineering]({{ '/01-databricks-fundamentals/02-introduction/introduction-to-data-engineering/' | relative_url }}) | [Next: Introduction to Databricks Platform &rarr;]({{ '/01-databricks-fundamentals/02-introduction/introduction-to-databricks-platform/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
