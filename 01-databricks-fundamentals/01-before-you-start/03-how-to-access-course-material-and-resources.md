---
title: "How to access Course Material and Resources"
parent: "Before you start"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/01-before-you-start/how-to-access-course-material-and-resources/
read_minutes: 2
---

# How to access Course Material and Resources
{: .no_toc }

*Estimated read: 2 min*

There's no video player and no separate download portal for this guide -- everything lives on the
page you're reading, as **Markdown** with syntax-highlighted code blocks, tables, and diagrams. A
few things to know about how the material is organized and where to go for the parts we don't
reproduce for you.

## Navigating the site

- The **sidebar** on the left (powered by the `just-the-docs` theme) mirrors the full hierarchy:
  Part -> Section -> Lecture. It's present on every page.
- The [Course Map]({{ '/course-map/' | relative_url }}) is the single-page version of that same hierarchy -- every part,
  section, and lecture linked from one scrollable page, set in a distinct serif typeface so it
  reads as an overview, not a lesson. Bookmark it if you like jumping around by topic rather than
  reading start to finish.
- Every page ends with **Previous / Next** links that walk the entire course in sequence, so you
  can also just keep clicking "Next" from the [Home page]({{ '/' | relative_url }}) and never touch the sidebar.

## Code and diagrams

Every code example in this guide is a real, runnable snippet -- PySpark, Spark SQL, Databricks
CLI, or YAML for asset bundles -- shown in a fenced code block you can copy directly into a
notebook cell. **Key terms** are bolded on first use, and an `.important` callout box, used
sparingly, flags the one gotcha per lecture that actually costs people time in production.

Sections that involve a multi-step architecture or data flow (Medallion layers, Unity Catalog's
object hierarchy, a migration cutover sequence) include a **Mermaid diagram** at the top of the
section, rendered directly in the page -- no separate image files to keep in sync.

## Official Databricks resources referenced throughout

Where the official documentation or an official demo directly supports what a lecture is
teaching, this guide links to it rather than re-explaining what Databricks already documents well.
The two you'll see most:

- [Databricks documentation](https://docs.databricks.com/aws/en/introduction/) -- the canonical
  reference for every feature covered here, kept current by Databricks itself.
- [Databricks Industry Solutions](https://github.com/databricks-industry-solutions) on GitHub --
  official, fully-functional solution accelerator notebooks maintained by Databricks, useful once
  you want to see a pattern from this guide applied at real-world scale.

## The StepRight project (Part 2)

Part 2 is a single project built incrementally across eight sections. Its starter data generator
and project skeleton are introduced in that part's first section
(**StepRight - Project Foundation**) rather than here -- you won't need it until you're done with
Part 1.

<!-- prevnext:start -->

---

| [&larr; Previous: Course Prerequisites]({{ '/01-databricks-fundamentals/01-before-you-start/course-prerequisites/' | relative_url }}) | [Next: Introduction &rarr;]({{ '/01-databricks-fundamentals/02-introduction/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
