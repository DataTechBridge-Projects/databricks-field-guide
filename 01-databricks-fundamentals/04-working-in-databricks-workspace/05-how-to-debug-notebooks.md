---
title: "How to Debug Notebooks"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/how-to-debug-notebooks/
read_minutes: 5
---

# How to Debug Notebooks
{: .no_toc }

*Estimated read: 5 min*

Cell-by-cell execution already gives you a form of debugging a linear SQL script doesn't -- run
up to the point of failure, inspect what's actually in a DataFrame, adjust, rerun just that cell.
This lecture covers the two levels beyond that: quick data inspection, and the full interactive
debugger for genuinely tricky logic bugs.

## Level 1: inspect the data directly

Before reaching for a debugger, the fastest diagnostic is almost always looking at the data itself:

```python
df.show(20, truncate=False)       # print rows directly
df.printSchema()                  # confirm column types match expectations
df.count()                        # sanity-check row counts at each stage
display(df)                       # Databricks' rich, interactive table/chart view
```

`display()` is Databricks-specific and worth defaulting to over `.show()` in a notebook you're
actively debugging -- it renders a sortable, scrollable table and can switch straight to a chart
view, closer to browsing results in a SQL client than reading raw console output.

## Level 2: the interactive Python debugger

For a genuine logic bug -- wrong values propagating through several transformation steps, an
exception buried inside a UDF -- Databricks notebooks support an interactive debugger similar to
`pdb` or an IDE breakpoint, directly inline:

1. **Attach the notebook to a cluster** (the debugger requires an active, running cluster --
   it won't work against a detached notebook).
2. Set a **breakpoint** by clicking the line-number gutter next to the line you want execution to
   pause at, or insert `breakpoint()` directly in code.
3. Run the cell. Execution pauses at the breakpoint, and a debugger panel opens showing local
   variables at that point in execution.
4. Step through line by line (**step over**, **step into**, **continue**), inspecting variables --
   including calling `df.show()` from *within* the paused debugger session to check a DataFrame's
   actual contents at that exact point.

```python
def compute_discount(order_total, discount_pct):
    breakpoint()  # execution pauses here when this line runs
    return order_total * (1 - discount_pct)
```

**Key term:** this is a genuine **interactive debugger** attached to the live Python process
running your cell -- not a print-statement-driven approximation. It's the closest equivalent to
stepping through a stored procedure with a debugger attached, if your legacy tooling ever supported
that.

## When to reach for which level

| Situation | Use |
|---|---|
| "Are the values roughly what I expect?" | `display()` / `.show()` |
| "Did the schema change unexpectedly?" | `.printSchema()` |
| "Row counts look wrong somewhere in a multi-step pipeline" | `.count()` at each stage |
| "A specific calculation is silently producing wrong results" | Interactive debugger, breakpoint at the calculation |
| "An exception is thrown but the stack trace doesn't make it obvious why" | Interactive debugger, breakpoint before the failing line |

For most day-to-day pipeline development, `display()` plus careful row-count checks at each
Medallion layer boundary catches the majority of issues before you need the full debugger at all.
{: .important }

<!-- prevnext:start -->

---

| [&larr; Previous: Databricks Utilities and Widgets]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-utilities-and-widgets/' | relative_url }}) | [Next: Workspace Files vs Git Folders &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/workspace-files-vs-git-folders/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
