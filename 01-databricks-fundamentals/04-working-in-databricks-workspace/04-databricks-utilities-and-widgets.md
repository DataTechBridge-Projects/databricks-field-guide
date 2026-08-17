---
title: "Databricks Utilities and Widgets"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 4
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-utilities-and-widgets/
read_minutes: 20
---

# Databricks Utilities and Widgets
{: .no_toc }

*Estimated read: 20 min*

**`dbutils`** is a Python (and Scala/R) API, available automatically in every Databricks notebook,
that covers everything Spark's own DataFrame API doesn't: filesystem operations, secrets,
parameterization, and cross-notebook control flow. If `%fs` and `%run` from the previous lecture
felt useful, this lecture is their full underlying API, plus the pieces that don't have a magic-
command shortcut at all. See the official
[Databricks Utilities documentation](https://docs.databricks.com/aws/en/dev-tools/databricks-utils)
for the complete, current API reference across all languages.

## `dbutils.fs`: the full filesystem API

`%fs` commands are shorthand for `dbutils.fs` calls -- the full API gives you programmatic access,
useful inside actual pipeline code rather than ad hoc checks:

```python
dbutils.fs.ls("/Volumes/main/landing/orders/")
dbutils.fs.mkdirs("/Volumes/main/landing/orders/2026-08-14/")
dbutils.fs.cp("source_path", "target_path", recurse=True)
dbutils.fs.mv("source_path", "target_path")
dbutils.fs.rm("path/to/remove", recurse=True)
dbutils.fs.head("path/to/file.csv", maxBytes=1000)  # peek without loading a full DataFrame
```

A common pattern: check whether a path already has data before deciding whether a pipeline step is
a first run or an incremental one --

```python
existing_files = dbutils.fs.ls("/Volumes/main/landing/orders/")
is_first_run = len(existing_files) == 0
```

## `dbutils.widgets`: parameterizing notebooks

**Widgets** turn a notebook into something you can run with different inputs without editing code
-- the direct equivalent of parameters you'd pass into a Talend job or a stored procedure call.

```python
dbutils.widgets.text("run_date", "2026-08-14", "Run Date")
dbutils.widgets.dropdown("environment", "dev", ["dev", "uat", "prod"], "Environment")
dbutils.widgets.combobox("region", "us-east-1", ["us-east-1", "us-west-2", "eu-west-1"], "Region")
dbutils.widgets.multiselect("categories", "electronics", ["electronics", "apparel", "home"], "Categories")

run_date = dbutils.widgets.get("run_date")
environment = dbutils.widgets.get("environment")
```

| Widget type | Use case |
|---|---|
| `text` | Free-form input -- a run date, a customer ID for debugging |
| `dropdown` | A fixed, small set of choices -- environment, region |
| `combobox` | A suggested set of choices that also accepts free text |
| `multiselect` | Multiple values from a fixed set -- categories to process |

When a notebook is attached to a cluster interactively, widgets render as actual UI controls at
the top of the notebook -- a teammate can change `environment` from a dropdown without touching
code. When the same notebook runs as a **Lakeflow Jobs task** (Section 10), those same widget
values are supplied as **job parameters** instead -- the same parameterization mechanism serving
both interactive exploration and scheduled production runs.
{: .important }

## `dbutils.secrets`: credentials without hardcoding

```python
db_password = dbutils.secrets.get(scope="prod-db-creds", key="password")
```

Secrets are stored in a **secret scope** (backed by Databricks' own secret manager, or an external
one like AWS Secrets Manager) and retrieved by scope and key -- never hardcoded in a notebook, and
never printed to notebook output; Databricks automatically redacts a secret's value if a cell
tries to print it. This replaces whatever pattern you previously used for credential management in
a legacy ETL tool -- an encrypted properties file, a vault integration, or (worse) a value pasted
directly into a job definition.

## `dbutils.notebook`: chaining notebooks with control flow

Where `%run` executes another notebook *inline*, sharing the current session, `dbutils.notebook.run`
executes another notebook as a **separate job run**, on its own cluster resources, and can pass
parameters in and receive a return value back:

```python
result = dbutils.notebook.run(
    "/pipelines/bronze_orders",
    timeout_seconds=600,
    arguments={"run_date": run_date, "environment": environment}
)
```

Inside the called notebook, it ends by returning a value with `dbutils.notebook.exit`:

```python
# At the end of bronze_orders notebook
dbutils.notebook.exit(f"processed {row_count} rows")
```

**`%run` vs. `dbutils.notebook.run`, the distinction that matters:** `%run` shares your session and
variables (a Python `import`, effectively); `dbutils.notebook.run` runs the target notebook
independently and communicates only through parameters in and a single return value out (closer to
calling a separate job). Use `%run` for shared setup code within one logical pipeline; use
`dbutils.notebook.run` when you genuinely need isolation between steps -- though for production
orchestration, Lakeflow Jobs (Section 10) is generally the better tool for chaining notebooks with
retries, monitoring, and scheduling built in, rather than nesting `dbutils.notebook.run` calls by
hand.

## `dbutils.library` and other utilities

Less frequently used, but worth knowing exist: `dbutils.library` for managing libraries installed
on a cluster from within a notebook (largely superseded by cluster-level library configuration and
`%pip install`), and `dbutils.data.summarize(df)` for a quick statistical profile of a DataFrame --
useful during initial data exploration, comparable to running `DESCRIBE` plus a set of ad hoc
aggregate queries in a SQL client, condensed into one call.

## A realistic combined example

```python
dbutils.widgets.text("run_date", "2026-08-14", "Run Date")
run_date = dbutils.widgets.get("run_date")

source_path = f"/Volumes/main/landing/orders/{run_date}/"
existing = dbutils.fs.ls(source_path) if _path_exists(source_path) else []

db_password = dbutils.secrets.get(scope="prod-db-creds", key="password")

if not existing:
    dbutils.notebook.exit(f"No files found for {run_date}, skipping")

df = spark.read.format("json").load(source_path)
row_count = df.count()
dbutils.notebook.exit(f"Loaded {row_count} rows for {run_date}")
```

This one snippet uses widgets for parameterization, `fs` for a landing-zone check, `secrets` for a
downstream credential, and `notebook.exit` to report back to whatever called it -- the four
`dbutils` submodules you'll reach for most across the rest of this guide.

<!-- prevnext:start -->

---

| [&larr; Previous: Notebook Magic Commands]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/notebook-magic-commands/' | relative_url }}) | [Next: How to Debug Notebooks &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/how-to-debug-notebooks/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
