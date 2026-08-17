---
title: "Course Prerequisites"
parent: "Before you start"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/01-before-you-start/course-prerequisites/
read_minutes: 3
---

# Course Prerequisites
{: .no_toc }

*Estimated read: 3 min*

This guide assumes you can already write SQL comfortably and have built or maintained at least one
production ETL pipeline, in any tool. It does **not** assume you already know Apache Spark or
Databricks -- that's what Part 1 teaches. Here's exactly where the floor is.

## What you need coming in

- **SQL, at a working level.** Joins, window functions, CTEs, aggregations. If you've written a
  gnarly Oracle `MERGE` statement or a Teradata multi-join ETL query, you're well past this bar.
- **Python, at a basic level.** You don't need to be a software engineer. You need to be
  comfortable reading a function, understanding a `for` loop, and following a script that calls a
  few library functions. Most Databricks code in this guide is **PySpark**, which reads close to
  plain Python with SQL-shaped method names (`.filter()`, `.groupBy()`, `.join()`).
- **General ETL/ELT concepts.** Incremental vs. full loads, staging tables, idempotency, schema
  drift -- if you've had to explain to someone why a rerun of last night's job produced duplicate
  rows, you already understand the problems Delta Lake and Lakeflow solve. This guide will show you
  the Databricks-native vocabulary for problems you've already debugged by hand.

## What you do not need

- **No prior Apache Spark experience.** Part 1 introduces Spark **DataFrames**, transformations,
  and Spark SQL from the ground up, specifically framed against the ETL tools you already know.
- **No prior AWS expertise beyond basic familiarity.** You should know what an S3 bucket and an
  IAM user are; you don't need to have run production workloads on AWS. Section 3 of Part 1 walks
  through account and workspace setup end to end.
- **No Databricks account yet.** You'll create one in the next two sections. Databricks offers a
  genuinely useful no-cost tier -- see below -- so you can follow every hands-on example without a
  credit card.

## Setting up your own environment

Two options, covered in the next two sections of this part:

1. **Databricks Free Edition** -- a free, serverless workspace, sufficient for all of Part 1 and
   most of Part 2. See [Databricks Free Edition](https://www.databricks.com/learn/free-edition) for
   what's included.
2. **Databricks on AWS (paid, marketplace-billed)** -- needed if you want to follow the AWS-account
   setup path in Section 3, or if you're specifically preparing for the migration content in Part 3,
   which assumes a real workspace tied to your own cloud account.

If you want to preview the Spark APIs you'll be using before you touch Databricks at all, the
official [Apache Spark documentation](https://spark.apache.org/docs/latest/) covers the Quick
Start and the SQL/DataFrame programming guide referenced throughout Part 1.

<!-- prevnext:start -->

---

| [&larr; Previous: About the Course]({{ '/01-databricks-fundamentals/01-before-you-start/about-the-course/' | relative_url }}) | [Next: How to access Course Material and Resources &rarr;]({{ '/01-databricks-fundamentals/01-before-you-start/how-to-access-course-material-and-resources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
