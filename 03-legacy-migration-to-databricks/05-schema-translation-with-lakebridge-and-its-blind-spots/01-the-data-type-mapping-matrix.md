---
title: "The Data Type Mapping Matrix"
parent: "Schema Translation with Lakebridge (and Its Blind Spots)"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/the-data-type-mapping-matrix/
read_minutes: 3
---

# The Data Type Mapping Matrix
{: .no_toc }

*Estimated read: 3 min*

Every schema translation project starts with the same deceptively simple-looking task: mapping
source column types to Delta types. It looks like a lookup table exercise. In practice it's where
the majority of the quiet, hard-to-detect migration bugs originate, because SQL is forgiving about
type mismatches in ways that only surface downstream -- a report that's off by a few cents, a join
that silently drops rows, a date that's one day early depending on the reader's timezone.

**Delta Lake's type system is deliberately smaller than any of the legacy engines it replaces.**
There's no `VARCHAR2(4000 BYTE)` vs `VARCHAR2(4000 CHAR)` distinction, no unconstrained `NUMBER`,
no engine-specific temporal type with its own rounding rules -- just `STRING`, `DECIMAL(p,s)`,
`INT`/`BIGINT`/`SMALLINT`/`TINYINT`, `DOUBLE`/`FLOAT`, `DATE`, `TIMESTAMP`/`TIMESTAMP_NTZ`,
`BOOLEAN`, `BINARY`, and the nested `ARRAY`/`MAP`/`STRUCT` types. Translating into that smaller
alphabet means some source types map cleanly and some require a judgment call.

| Source type | Common source engines | Delta mapping | Gotcha |
|---|---|---|---|
| `VARCHAR2(n)` / `VARCHAR(n)` | Oracle, Teradata, SQL Server | `STRING` | Length constraint is dropped -- `STRING` is unbounded. Enforce length with a `CHECK` constraint if the source relied on it for validation. |
| `NUMBER` (no precision/scale) | Oracle | `DECIMAL(38,10)` (convention) or `DOUBLE` | Unconstrained `NUMBER` is ambiguous by design. Widening to a fixed-precision `DECIMAL` is the safer default; falling back to `DOUBLE` risks silent precision loss (next lecture). |
| `NUMBER(p,s)` | Oracle, Teradata | `DECIMAL(p,s)` | Clean 1:1 mapping when precision and scale are both declared -- the case you want, and should push source teams toward before conversion. |
| `DATE` | Oracle | `TIMESTAMP` | Oracle's `DATE` always stores a time component down to the second. Mapping it to Delta `DATE` truncates that silently -- one of the single most common translation bugs in Oracle migrations. |
| `DATETIME2` | SQL Server | `TIMESTAMP` | Fractional-second precision (up to 7 digits in SQL Server) is wider than some downstream systems expect; confirm consumers don't depend on sub-microsecond precision. |
| `MONEY` / `SMALLMONEY` | SQL Server | `DECIMAL(19,4)` | Preserve the fixed 4-digit scale explicitly -- don't let it fall through to a default-precision `DECIMAL`. |
| `BYTEINT` | Teradata | `TINYINT` | Straightforward, but check for any source constraint logic that assumed Teradata's signed range. |
| `PERIOD(DATE)` / `PERIOD(TIMESTAMP)` | Teradata | No native equivalent -- decompose to two columns (`STRUCT` or `start`/`end`) | There is nothing in the Delta type system that models a temporal interval as a single column; this always requires a schema redesign decision, not a mechanical conversion. |
| `SDO_GEOMETRY` / `GEOGRAPHY` | Oracle Spatial, SQL Server | `STRING` (WKT) or `BINARY` (WKB) + `ST_*` functions | Covered in depth in the "hard ones" lecture later in this section. |

The pattern across nearly every gotcha row above is the same one you'll see again in the next
lecture: **Lakebridge's converter makes a defensible default choice, and the default is not always
the choice you want for a specific column.** Treat the generated DDL as a strong first draft that a
human reviews against source semantics -- not a finished migration.

{: .important }
> The Oracle `DATE`-includes-time-component gotcha is worth memorizing on its own: it's the single
> type-mapping mistake most likely to pass code review unnoticed, because `DATE` *looks* like an
> obviously safe mapping to Delta `DATE`, and the resulting bug (timestamps silently truncated to
> midnight) often doesn't surface until a downstream report's hourly breakdown goes empty.

Lakebridge's own [type-mapping conventions](https://databrickslabs.github.io/lakebridge/docs/transpile/)
are documented per source dialect inside its transpiler configuration -- worth reading end-to-end
once for whichever source system you're converting from, rather than discovering the defaults one
column at a time in production.

<!-- prevnext:start -->

---

| [&larr; Previous: Schema Translation with Lakebridge (and Its Blind Spots)]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/' | relative_url }}) | [Next: Running Lakebridge's Analyzer and Converter on Real DDL &rarr;]({{ '/03-legacy-migration-to-databricks/05-schema-translation-with-lakebridge-and-its-blind-spots/running-lakebridges-analyzer-and-converter-on-real-ddl/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

