---
title: "Designing the Silver Layer"
parent: "Medallion Architecture - Design and Implementation"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-silver-layer/
read_minutes: 12
---

# Designing the Silver Layer
{: .no_toc }

*Estimated read: 12 min*

Silver is where "is this data trustworthy" actually gets decided -- the layer that does the work a
legacy warehouse's staging-to-integration transformation logic used to do, made explicit and
auditable rather than buried in a long ETL script.

## Silver's responsibilities

- **Typing and casting**, with validation -- a string that should be a date either becomes a valid
  `DATE` or is flagged/rejected, not silently coerced into something wrong.
- **Deduplication** -- the responsibility explicitly deferred from bronze. Typically via `MERGE
  INTO` (Section 5) keyed on a natural or surrogate key, or `ROW_NUMBER()`-based dedup logic before
  a merge.
- **Business rule application** -- validating referential integrity, range checks, required-field
  checks, the specific rules that define "correct" for your domain.
- **Conforming to a stable schema** -- unlike bronze's flexible/`VARIANT` shape, silver tables have
  a well-defined, enforced schema (Section 5's schema enforcement, used deliberately here).

```sql
CREATE TABLE silver.orders (
    order_id       BIGINT NOT NULL,
    customer_id    BIGINT NOT NULL,
    order_total    DECIMAL(10,2),
    order_status   STRING,
    order_date     DATE,
    _quality_flag  STRING  -- 'valid' | 'quarantined', from data quality rules
)
USING DELTA;
```

## Report-only vs. quarantine: two validation strategies

Not every data quality issue should block a pipeline. Two common patterns, often combined:

| Strategy | Behavior | Use for |
|---|---|---|
| **Report-only** | Row still flows through silver, flagged with a quality issue for monitoring | Minor issues, informational anomalies |
| **Quarantine** | Row is diverted to a separate quarantine table, excluded from the main silver flow | Critical issues -- a null required field, a referential integrity violation |

```sql
-- Report-only: keep the row, flag it
INSERT INTO silver.orders
SELECT *, CASE WHEN order_total < 0 THEN 'flagged_negative_total' ELSE 'valid' END AS _quality_flag
FROM bronze.orders;

-- Quarantine: route bad rows to a separate table entirely
INSERT INTO silver.orders_quarantine
SELECT * FROM bronze.orders WHERE customer_id IS NULL;
```

**Key term:** this exact **report-only-by-default, selective quarantine for critical issues**
pattern is what
[Part 2's StepRight project builds fully]({{ '/02-stepright-capstone-project/03-stepright-transformation-layers/' | relative_url }})
in its transformation layer section, with twelve concrete business rules across four categories --
worth previewing there once you're ready to see this design decision implemented end to end.
{: .important }

## SCD Type 2 at the silver layer

For dimension-shaped data (customers, products) where **history of change matters** -- not just
current state -- silver is where **SCD Type 2** modeling typically lives, using the `MERGE INTO`
pattern from Section 5:

```sql
-- Simplified SCD2 pattern: close out changed rows, insert new current version
MERGE INTO silver.customers AS target
USING (SELECT *, current_timestamp() AS effective_ts FROM bronze.customers_dedup) AS source
ON target.customer_id = source.customer_id AND target.is_current = true
WHEN MATCHED AND target.email != source.email THEN
  UPDATE SET target.is_current = false, target.ended_at = source.effective_ts
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, is_current, started_at, ended_at)
  VALUES (source.customer_id, source.email, true, source.effective_ts, NULL);
```

## What silver deliberately doesn't do

Silver conforms and validates -- it doesn't aggregate for a specific business consumer. A
"daily revenue by region" table is a **gold** concern, built *from* silver, not computed inside
silver itself. Keeping this boundary clean is what lets multiple, differently-shaped gold tables
all derive from the same trustworthy silver layer, rather than each consumer's aggregation logic
getting tangled into the validation logic everyone depends on.
{: .important }

<!-- prevnext:start -->

---

| [&larr; Previous: Designing the Bronze Layer]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-bronze-layer/' | relative_url }}) | [Next: Designing the Gold Layer &rarr;]({{ '/01-databricks-fundamentals/07-medallion-architecture-design-and-implementation/designing-the-gold-layer/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
