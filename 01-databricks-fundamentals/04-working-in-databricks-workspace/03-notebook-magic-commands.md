---
title: "Notebook Magic Commands"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/notebook-magic-commands/
read_minutes: 11
---

# Notebook Magic Commands
{: .no_toc }

*Estimated read: 11 min*

**Magic commands** are the `%`-prefixed directives that make a Databricks notebook more than "a
Python file split into cells" -- they switch languages, touch the filesystem, and let notebooks
call each other. This lecture covers the ones you'll use constantly.

## Language-switching magics

Every notebook has a default language, but any cell can override it for that cell only:

```text
%python   -- run this cell as Python, regardless of notebook default
%sql      -- run this cell as Spark SQL
%scala    -- run this cell as Scala
%r        -- run this cell as R
%md       -- render this cell as Markdown (documentation, not code)
```

```sql
%sql
-- Even in a Python-default notebook, this cell runs as SQL,
-- and can reference temp views created by earlier Python cells.
SELECT customer_id, sum(order_total) AS revenue
FROM orders_temp_view
GROUP BY customer_id
ORDER BY revenue DESC
LIMIT 10
```

**Key term:** all cells in a notebook share the same underlying **Spark session**, regardless of
which language each cell is written in -- a temp view created in one language is immediately
queryable from any other. This is different from most legacy ETL tools, where a Python step and a
SQL step were genuinely separate executions with data passed via a staged table.

## `%fs`: filesystem operations

```text
%fs ls /Volumes/main/default/landing/
%fs mkdirs /Volumes/main/default/landing/orders/
%fs cp source_path target_path
```

`%fs` is shorthand for the file-system portion of `dbutils.fs` (covered in full in the
**Databricks Utilities and Widgets** lecture) -- useful for quick, ad hoc checks of what's actually
in a volume or bucket without writing a full `dbutils.fs.ls(...)` call.

## `%run`: modular notebooks

```text
%run ./shared/common_functions
```

`%run` executes another notebook *in the current notebook's session* -- any functions, variables,
or temp views it defines become available afterward, as if you'd pasted its contents in. This is
the primary way to build a **modular workflow**: shared setup logic, common transformation
functions, or configuration lives in one notebook, and every notebook that needs it starts with a
`%run` at the top.

```text
Project structure using %run:

  common/
    setup.py            <- catalog/schema config, shared constants
    transformations.py  <- reusable transformation functions

  pipelines/
    bronze_orders.py     %run ../common/setup
                          %run ../common/transformations
                          ... pipeline-specific logic
```

**Compared to your ETL background:** `%run` plays a role similar to a shared "common" module or
include file in a legacy ETL tool -- except the include happens at notebook-run time, in the same
session, not as a separately packaged and deployed library. For genuinely reusable, tested code
shared across many pipelines (rather than one project's internal modularity), a proper Python
package is usually the better long-term choice -- something Part 2's StepRight project addresses
directly once the project grows past a handful of notebooks.
{: .important }

## Other useful magics

- `%pip install <package>` -- install a Python package scoped to the current notebook session,
  without needing to pre-bake it into a cluster image.
- `%sh` -- run a shell command on the driver node -- useful for quick diagnostics, rarely something
  you'd build production logic around.

## Putting it together

A realistic notebook header combining several of these:

```text
%md
# Bronze Ingestion — Orders

%run ./common/setup

%fs ls /Volumes/main/landing/orders/

%python
df = spark.read.format("json").load("/Volumes/main/landing/orders/")
```

<!-- prevnext:start -->

---

| [&larr; Previous: Introduction to Databricks Notebooks]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/introduction-to-databricks-notebooks/' | relative_url }}) | [Next: Databricks Utilities and Widgets &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-utilities-and-widgets/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
