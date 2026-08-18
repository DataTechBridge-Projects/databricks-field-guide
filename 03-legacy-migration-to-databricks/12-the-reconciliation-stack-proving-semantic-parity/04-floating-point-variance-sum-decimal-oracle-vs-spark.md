---
title: "Floating-Point Variance: SUM(DECIMAL) Oracle vs Spark"
parent: "The Reconciliation Stack: Proving Semantic Parity"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/floating-point-variance-sum-decimal-oracle-vs-spark/
read_minutes: 3
---

# Floating-Point Variance: SUM(DECIMAL) Oracle vs Spark
{: .no_toc }

*Estimated read: 3 min*

Oracle's `NUMBER` type is famously permissive: it stores up to 38 significant digits with a scale that can float, and Oracle's arithmetic engine carries that variable-precision behavior through every intermediate step of a calculation. A migration that translates `NUMBER` columns into a Databricks column without specifying an explicit `DECIMAL(p, s)` -- or worse, into a `DOUBLE` -- inherits none of that behavior, and the divergence shows up exactly where a finance team looks first: `SUM()` over a large table of money values.

The mechanism is precision loss compounding across rows, not a single dramatic rounding error. `DOUBLE` is IEEE 754 binary floating point, which cannot represent most base-10 decimal fractions exactly -- `0.1` in binary floating point is actually a repeating fraction truncated to fit, the same reason `0.1 + 0.2` famously doesn't equal `0.3` exactly in most programming languages. A single row's tiny representation error is invisible. Summed across ten million order-line rows, those sub-cent errors accumulate into a total that's off from Oracle's `NUMBER`-based sum by real money -- sometimes tens of dollars, sometimes more, depending on the value distribution and row count. Layer 1 (count) and Layer 4 (row-level hash, if the hash is computed on rounded display values) can both pass clean while Layer 2 (sum) fails, which is exactly why sum parity is its own rung in the stack rather than something inferred from a passing checksum.

Databricks SQL's [`DECIMAL` type](https://docs.databricks.com/aws/en/sql/language-manual/data-types/decimal-type) is the fix, because it is fixed-point rather than floating-point -- it stores an exact base-10 value up to the declared precision and scale, with no binary-fraction approximation at all:

```sql
-- Wrong: DOUBLE silently accumulates binary floating-point error
CREATE TABLE silver.order_lines (
  order_line_id BIGINT,
  unit_price     DOUBLE,   -- avoid for money
  quantity       INT
);

-- Right: DECIMAL mirrors Oracle NUMBER(p,s) with fixed, exact scale
CREATE TABLE silver.order_lines (
  order_line_id BIGINT,
  unit_price     DECIMAL(12, 2),
  quantity       INT
);
```

The rule for migrating any Oracle `NUMBER(p, s)` column is to map it to an explicit `DECIMAL(p, s)` with matching precision and scale, never to `DOUBLE` or `FLOAT`, and never to a bare `DECIMAL` that falls back to the default `DECIMAL(10, 0)` -- which silently truncates any fractional cents. A `NUMBER` column with no declared precision or scale in Oracle (a genuinely floating-precision value) needs a deliberate scale decision during schema translation, not an automatic pass-through, because Spark has no equivalent "figure it out per value" numeric type.

{: .important }
> Never sum money as `DOUBLE`. Even when a single query's output looks correct to two decimal places, the underlying binary representation is not exact, and the error compounds silently across every subsequent aggregate built on top of that column -- by the time it surfaces in a quarterly report, it's a much harder number to trace back to its source.

Even with every column correctly typed as `DECIMAL`, some genuine rounding variance between Oracle's arithmetic and Spark's arithmetic can remain at the level of fractions of a cent per aggregate -- rounding-mode differences in intermediate division steps, for instance. The next lecture covers how to set a **tolerance threshold** that accepts that residual, expected variance without either rubber-stamping a real bug or chasing an unfixable rounding difference to exact zero.

<!-- prevnext:start -->

---

| [&larr; Previous: The $4M Bug: ROW_NUMBER Sort-Order Differences]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/the-4m-bug-row-number-sort-order-differences/' | relative_url }}) | [Next: Tolerance Thresholds: Acceptable vs Real Variance &rarr;]({{ '/03-legacy-migration-to-databricks/12-the-reconciliation-stack-proving-semantic-parity/tolerance-thresholds-acceptable-vs-real-variance/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

