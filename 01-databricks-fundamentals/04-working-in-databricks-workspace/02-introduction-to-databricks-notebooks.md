---
title: "Introduction to Databricks Notebooks"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/introduction-to-databricks-notebooks/
read_minutes: 9
---

# Introduction to Databricks Notebooks
{: .no_toc }

*Estimated read: 9 min*

A **notebook** is where you'll write nearly all the code in this guide -- the Databricks
equivalent of a SQL client's query window, but cell-based, multi-language, and collaborative by
default.

## Cells and execution

A notebook is a sequence of **cells**, each independently runnable. Run a cell (`Shift+Enter`, or
the run button) and its output -- a table preview, a plot, a print statement -- renders directly
below it. Unlike a linear SQL script, cells can be run out of order, which is powerful for
iterative development and a genuine footgun if you forget which cells you've actually run recently
in what order. **Run All** re-executes top to bottom when you need to confirm the notebook works
as a coherent whole, not just as whatever cells you happened to run interactively.
{: .important }

## Multi-language cells

Every cell has a **default language** for the notebook, but you can override it per cell with a
**magic command** -- `%sql`, `%python`, `%scala`, `%md` -- covered in full in the next lecture. This
means a single notebook can mix a Python transformation, a SQL validation query, and a Markdown
explanation, without switching tools or files.

```python
# Default language: Python
df = spark.table("bronze.orders")
```

```sql
-- A %sql cell in the same notebook, same session, same tables
SELECT count(*) FROM bronze.orders WHERE order_status = 'cancelled'
```

## Comparing to a legacy SQL client

If your prior workflow was a SQL client plus a separate script editor for any Python/ETL glue
code, the shift here is that both live in one document, with shared session state -- a DataFrame
created in a Python cell is queryable from a SQL cell in the same notebook (once registered as a
temp view) without exporting or re-importing anything.

## Collaboration and versioning

Multiple people can co-edit a notebook in real time, with cursors and comments visible live --
closer to a shared document than a checked-out script file. Databricks also tracks automatic
**revision history** independent of any Git integration, so you can review or roll back recent
changes even before you've committed anything (Git folders, the version-controlled alternative,
are covered later in this section).

## Attaching compute

A notebook does nothing until it's **attached** to compute -- a cluster or serverless compute, the
subject of this section's final lecture. The compute picker sits at the top of the notebook; until
something is attached and running, cells queue rather than execute. This attach/detach model is
the main structural difference from a warehouse SQL client, where a session is implicitly
"attached" to the engine the moment you connect.

For the full official reference -- including the AI-assisted coding features and dashboard-from-
notebook capabilities only briefly touched on here -- see
[Databricks notebooks documentation](https://docs.databricks.com/aws/en/notebooks/).

<!-- prevnext:start -->

---

| [&larr; Previous: Introduction to Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/introduction-to-databricks-workspace/' | relative_url }}) | [Next: Notebook Magic Commands &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/notebook-magic-commands/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
