---
title: "Introduction to Databricks Workspace"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 1
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/introduction-to-databricks-workspace/
read_minutes: 8
---

# Introduction to Databricks Workspace
{: .no_toc }

*Estimated read: 8 min*

You've created a workspace in the previous section (Free Edition or AWS-based). This lecture is a
tour of what's actually inside it -- the browser-based environment where every notebook, cluster,
job, and catalog object you touch for the rest of this guide lives.

## The workspace sidebar

The left-hand sidebar is your primary navigation, organized roughly the way a warehouse client's
object browser was, but spanning far more than tables:

- **Workspace** -- your folder tree of notebooks, files, and Git folders (covered in depth later
  in this section).
- **Catalog** -- the Unity Catalog browser: catalogs, schemas, tables, and volumes, with a
  built-in data preview, comparable to expanding a schema in a SQL client but with lineage and
  permissions visible inline.
- **Workflows** -- where Lakeflow Jobs live, covered in Section 10.
- **Compute** -- cluster management, covered later in this section.
- **SQL** -- SQL editor, dashboards, and (on paid tiers) SQL warehouses, a dedicated compute type
  optimized for BI-style queries rather than general Spark workloads.

## Folders, permissions, and organization

Workspace folders behave like a filesystem with **access control lists** attached at any level --
a folder can be shared with a specific user, a group, or made workspace-wide, and permissions
inherit downward unless explicitly overridden. This is a finer-grained model than a typical
warehouse's schema-level GRANT, closer to how a modern source control system scopes repository
access.

**Key term:** a **workspace** is the single Databricks deployment you're inside right now -- not
to be confused with an **account** (which can contain multiple workspaces, e.g. dev/uat/prod) or a
**metastore** (the Unity Catalog layer that can span multiple workspaces). You'll see all three
terms used precisely and distinctly throughout this guide.

## The top bar and search

The top bar surfaces **global search** (across notebooks, tables, and jobs -- genuinely useful
once a workspace has hundreds of objects), your **user settings** (workspace and editor themes,
covered when you created your Free Edition account), and, on multi-workspace accounts, a
**workspace switcher**.

## Where you'll actually spend your time

For the rest of Part 1, expect to move between three places repeatedly: the **notebook editor**
(next lecture), the **Catalog browser** (once Unity Catalog is introduced in Section 6), and
**Workflows** (Section 10). Everything else in the sidebar becomes relevant progressively as this
guide covers it -- you don't need to understand the whole interface before writing your first line
of code.
{: .important }

<!-- prevnext:start -->

---

| [&larr; Previous: Working in Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }}) | [Next: Introduction to Databricks Notebooks &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/introduction-to-databricks-notebooks/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
