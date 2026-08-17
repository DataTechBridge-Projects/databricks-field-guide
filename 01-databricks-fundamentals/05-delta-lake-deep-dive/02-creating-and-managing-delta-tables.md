---
title: "Creating and Managing Delta tables"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/creating-and-managing-delta-tables/
read_minutes: 15
---

# Creating and Managing Delta tables
{: .no_toc }

*Estimated read: 15 min*

Three ways to create a Delta table cover almost every situation you'll hit, and one distinction --
**managed vs. external** -- determines who actually owns the underlying data files. Get this
lecture solid; every pipeline in Part 2 depends on it.

## Approach 1: `CREATE TABLE`, explicit schema

```sql
CREATE TABLE main.default.orders (
    order_id      BIGINT,
    customer_id   BIGINT,
    order_total   DECIMAL(10,2),
    order_status  STRING,
    order_date    DATE
)
USING DELTA;
```

The direct equivalent of a `CREATE TABLE` DDL statement you'd write against a warehouse -- explicit
columns, explicit types, empty table. `USING DELTA` is actually the default on Databricks and can
be omitted, but writing it explicitly documents intent for anyone reading the DDL later.

## Approach 2: `CREATE TABLE ... AS SELECT` (CTAS)

```sql
CREATE TABLE main.default.orders_2026 AS
SELECT * FROM main.default.orders
WHERE order_date >= '2026-01-01';
```

Creates a new table with a schema **inferred from the query result** -- useful for derived tables,
one-off snapshots, or bootstrapping a table from an existing source without hand-writing a schema.
This is the same pattern as a warehouse `CTAS`, and behaves identically in spirit.

## Approach 3: Spark DataFrame write

```python
(df.write
   .format("delta")
   .mode("overwrite")
   .saveAsTable("main.default.orders"))
```

The programmatic path -- write a DataFrame you've built through transformations directly as a
managed table. `mode` controls what happens if the table already exists: `overwrite` (replace
entirely), `append` (add rows), `errorifexists` (fail if present), or `ignore` (skip silently if
present).

## Managed vs. external tables

This distinction matters more than it looks like it should, because it determines what happens
when you `DROP TABLE`:

| | Managed table | External table |
|---|---|---|
| Data location | Databricks-controlled default storage | A location **you** specify (an S3 path) |
| `DROP TABLE` behavior | Deletes both metadata **and** underlying data files | Deletes only the metadata/catalog entry -- data files remain |
| Typical use | Tables Databricks fully owns the lifecycle of | Data that must survive independent of the Databricks table definition, or is shared with non-Databricks tools |

```sql
-- Managed: Databricks owns storage location and lifecycle
CREATE TABLE main.default.orders (...) USING DELTA;

-- External: you own the storage location explicitly
CREATE TABLE main.default.orders_external (...)
USING DELTA
LOCATION 's3://my-bucket/orders/';
```

**Key term:** dropping a **managed** table is destructive to the actual data, not just the catalog
entry -- the same way dropping a table in a warehouse where the engine owns its own storage
removes the underlying data permanently. Dropping an **external** table is closer to removing a
warehouse's linked-server or external-table definition: the pointer disappears, the underlying
data doesn't. Choosing wrong here is one of the more common ways to lose data accidentally.
{: .important }

## Which should you default to?

For most pipeline tables inside a governed catalog structure (the pattern this guide uses
throughout, and what Unity Catalog in Section 6 formalizes), **managed tables** are the right
default -- you want Databricks to own the full lifecycle, and Unity Catalog's external locations
(covered in Section 6) give you governed storage placement without needing every individual table
to be external. Reach for **external tables** specifically when data needs to persist independent
of any single table definition -- for example, a bronze landing location multiple pipelines read
from, or data a non-Databricks tool also needs direct access to.

## Reading a table's location and history

```sql
DESCRIBE DETAIL main.default.orders;
DESCRIBE HISTORY main.default.orders;
```

`DESCRIBE DETAIL` shows the table's actual storage location, format, and size -- useful for
confirming whether a table is managed or external without checking the DDL. `DESCRIBE HISTORY`
previews the transaction log itself as a queryable table -- the mechanism the next lecture's time
travel content builds on directly.

<!-- prevnext:start -->

---

| [&larr; Previous: What is Delta Lake and why it matters]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/what-is-delta-lake-and-why-it-matters/' | relative_url }}) | [Next: Reading and writing Delta - batch and streaming &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/reading-and-writing-delta-batch-and-streaming/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
