---
title: "Check Your Knowledge"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up in this section -- the workspace, notebooks, magic commands, `dbutils`,
debugging, Git folders, and compute clusters.

1. What is the primary difference between `%run` and `dbutils.notebook.run`?
   A. `%run` is faster
   B. `%run` shares the calling notebook's session and variables; `dbutils.notebook.run` executes the target notebook as a separate, isolated run
   C. `dbutils.notebook.run` only works with SQL notebooks
   D. There is no functional difference

2. Which magic command lets a cell in a Python-default notebook run as Spark SQL?
   A. `%python-sql`
   B. `%sql`
   C. `#!sql`
   D. `--sql`

3. What is a **jobs cluster**, compared to an all-purpose cluster?
   A. A cluster reserved exclusively for SQL warehouses
   B. A cluster created for a single scheduled job run and torn down immediately afterward
   C. A cluster that can never autoscale
   D. A cluster type only available on Azure

4. Where should credentials such as a database password be stored for use in a notebook?
   A. Hardcoded directly in the notebook cell
   B. In a plain text workspace file
   C. In a secret scope, retrieved via `dbutils.secrets.get`
   D. In a notebook widget

5. What does a **widget** created with `dbutils.widgets` become when the same notebook runs as a Lakeflow Jobs task?
   A. It is ignored entirely
   B. It becomes a job parameter, supplying the same value non-interactively
   C. It converts automatically into a secret
   D. It forces the job to run on a jobs cluster only

6. What is the main advantage of a **Git folder** over a plain workspace file for production pipeline code?
   A. Git folders run faster
   B. Git folders support branches, pull requests, and integrate with CI/CD, unlike workspace files' simple revision history
   C. Workspace files cannot be edited by more than one person
   D. Git folders don't require a cluster to run

7. What AWS cost-saving option can worker nodes (but generally not the driver node) safely use?
   A. Reserved Instances only
   B. Spot instances
   C. Dedicated hosts
   D. Free-tier instances only

8. What does an **auto-termination** setting on an all-purpose cluster do?
   A. Deletes all notebooks attached to the cluster
   B. Automatically shuts down the cluster after a period of inactivity, to avoid unnecessary cost
   C. Terminates the workspace itself
   D. Prevents the cluster from ever being restarted

9. What is a **cluster policy** used for?
   A. Encrypting data at rest
   B. Constraining what cluster configuration options a user is allowed to choose
   C. Defining Unity Catalog permissions
   D. Scheduling job runs

10. Which tool is best suited for compute optimized specifically for SQL/BI dashboard workloads?
    A. An all-purpose cluster
    B. A jobs cluster
    C. A SQL warehouse
    D. An instance pool

## Answer Key

1. **B** -- `%run` inlines the target notebook into the current session (shared variables); `dbutils.notebook.run` executes it as an independent run with parameters in and a return value out.
2. **B** -- `%sql` switches that cell's language to Spark SQL regardless of the notebook's default.
3. **B** -- a jobs cluster exists only for the duration of the scheduled run it was created for, then terminates, incurring no idle cost.
4. **C** -- secrets belong in a secret scope, retrieved via `dbutils.secrets.get`, never hardcoded or plain-text.
5. **B** -- the same parameterization mechanism serves both interactive widgets and non-interactive job parameters.
6. **B** -- branching, pull requests, and CI/CD integration are Git folder capabilities that plain workspace files don't have.
7. **B** -- spot instances offer significant savings for workers; losing the driver to reclamation fails the whole job, so it's generally avoided there.
8. **B** -- auto-termination shuts down an idle all-purpose cluster automatically, the main defense against accidental cost from a forgotten running cluster.
9. **B** -- cluster policies are admin-defined templates that restrict the configuration choices available to users.
10. **C** -- SQL warehouses are the compute type purpose-built for SQL/BI workloads, available in serverless or classic form.

<!-- prevnext:start -->

---

| [&larr; Previous: Databricks Compute Cluster]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-compute-cluster/' | relative_url }}) | [Next: Delta Lake - Deep Dive &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

