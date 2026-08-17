---
title: "Type Widening and the Variant Data Type"
parent: "Delta Lake - Deep Dive"
grand_parent: "Databricks Fundamentals"
nav_order: 10
permalink: /01-databricks-fundamentals/05-delta-lake-deep-dive/type-widening-and-the-variant-data-type/
read_minutes: 13
---

# Type Widening and the Variant Data Type
{: .no_toc }

*Estimated read: 13 min*

Two schema-related capabilities that go beyond the "add a column" evolution from the previous
lecture: **widening a column's type in place**, and **storing genuinely semi-structured data
without pre-defining a rigid schema for it** via the `VARIANT` type.

## Type widening

The previous lecture showed `ALTER TABLE ... ALTER COLUMN ... TYPE` for changing a column's type.
**Type widening** specifically refers to *safe*, information-preserving type changes -- ones where
every existing value fits losslessly into the new type:

```sql
ALTER TABLE main.default.orders ALTER COLUMN quantity TYPE BIGINT;   -- INT -> BIGINT
ALTER TABLE main.default.orders ALTER COLUMN unit_price TYPE DOUBLE; -- FLOAT -> DOUBLE
ALTER TABLE main.default.orders ALTER COLUMN order_total TYPE DECIMAL(12,4); -- DECIMAL(10,2) -> DECIMAL(12,4)
```

| Widening direction | Example |
|---|---|
| Smaller integer to larger | `INT -> BIGINT` |
| Lower to higher precision float | `FLOAT -> DOUBLE` |
| Smaller decimal to larger scale/precision | `DECIMAL(10,2) -> DECIMAL(12,4)` |

**Key term:** widening is a **metadata-only** operation when supported -- Delta doesn't need to
rewrite every existing data file, because every existing value already fits the new, larger type.
This is meaningfully cheaper than a full table rewrite, and is exactly the operation you'd reach
for when a legacy `INT` column turns out to need `BIGINT` range as data volume grows past what
anyone originally planned for.
{: .important }

Narrowing (the reverse direction -- `BIGINT` to `INT`, `DOUBLE` to `FLOAT`) is **not** supported as
an in-place operation, because it can lose data -- that requires an explicit rewrite with your own
validation that no values are actually out of range for the smaller type.

## The `VARIANT` type: semi-structured data, natively

Legacy warehouses generally forced a choice for JSON-shaped data: flatten it into rigid columns up
front (losing flexibility if the source schema shifts), or store it as a raw string column (losing
queryability -- every access needs a parse). **`VARIANT`** is Databricks' answer to that tradeoff:
a native column type that stores semi-structured data efficiently while remaining directly
queryable, without predefining its internal shape.

```sql
CREATE TABLE main.default.events (
    event_id BIGINT,
    payload  VARIANT
);

INSERT INTO main.default.events
SELECT 1, parse_json('{"type": "click", "meta": {"page": "/checkout", "duration_ms": 340}}');
```

```sql
-- Querying into a variant column
SELECT
  event_id,
  payload:type::STRING              AS event_type,
  payload:meta.page::STRING         AS page,
  payload:meta.duration_ms::INT     AS duration_ms
FROM main.default.events;
```

The `:` operator navigates into the variant, `.` descends into nested objects, and `::TYPE` casts
the extracted value -- `variant_get()` is the equivalent function form if you prefer that syntax.

## Why `VARIANT` over a plain JSON string column

| | JSON stored as `STRING` | `VARIANT` |
|---|---|---|
| Query without parsing | No -- every access needs `from_json`/manual parsing | Yes -- direct `:` path access |
| Schema flexibility | Full (it's just text) | Full (still no fixed schema required) |
| Storage/query efficiency | Poor -- reparsed on every query | Better -- Databricks stores it in an optimized internal format |
| Type safety on extraction | Manual | `::TYPE` casts, `try_variant_get` for safe casting |

## Real-world use in this guide's context

`VARIANT` is the natural landing type for **bronze-layer** data straight from an API or event
source, where the source schema isn't fully known or stable up front -- exactly the situation
[Lakeflow Connect's ingestion patterns]({{ '/01-databricks-fundamentals/08-lakeflow-connect-ingestion-pillar/' | relative_url }})
cover later in this part. Land the raw payload as `VARIANT` in bronze, then extract and type only
the fields silver-layer transformations actually need -- avoiding both the "guess the whole schema
up front" problem and the "everything is an unqueryable string" problem at once.

## Known limitations

`VARIANT` columns can't currently be used as **clustering keys**, **partition columns**, or
**Z-order keys**, and have some restrictions on direct comparison and grouping operations --
extract the specific fields you need to filter or cluster on into their own typed columns rather
than relying on the variant column itself for those operations.

For the complete official reference, including `variant_explode` for arrays and `VariantVal`'s
Python API, see [Query variant data](https://docs.databricks.com/aws/en/semi-structured/variant).

<!-- prevnext:start -->

---

| [&larr; Previous: Schema Enforcement and Schema Evolution]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/schema-enforcement-and-schema-evolution/' | relative_url }}) | [Next: Table constraints - NOT NULL, CHECK, and Identity Columns &rarr;]({{ '/01-databricks-fundamentals/05-delta-lake-deep-dive/table-constraints-not-null-check-and-identity-columns/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
