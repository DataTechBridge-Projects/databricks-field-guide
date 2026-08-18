---
title: "Syntactically Valid, Semantically Wrong"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/syntactically-valid-semantically-wrong/
read_minutes: 4
---

# Syntactically Valid, Semantically Wrong
{: .no_toc }

*Estimated read: 4 min*

The window-function swap in the previous lecture is one member of a much larger family: PySpark that an LLM produces, that runs cleanly against real data, that a code reviewer skimming for syntax errors would approve without a second thought -- and that computes a different answer than the Oracle or SQL Server procedure it replaced. None of these require a bug in the model's SQL knowledge. Each one is the model correctly applying Spark's default behavior where the source system's default behavior was different, because nothing in an underspecified prompt told it the difference mattered. Four of the most common are worth knowing by name before you meet them in your own output.

**Empty string versus NULL.** Oracle treats an empty string (`''`) as `NULL` for most purposes -- `column = ''` never matches anything, the same as `column = NULL` never matches. Spark treats `''` and `NULL` as genuinely distinct values. An LLM asked to port a `VARCHAR2` column that has, in practice, only ever held real strings or Oracle's NULL-as-empty-string will often translate the *type* faithfully while missing that any literal `''` values baked into the source data (or produced by an upstream `NVL(x, '')`) now behave like a real, non-null string in Spark -- `WHERE column IS NULL` silently stops matching rows it used to match.

**Blank-padded `CHAR` comparisons.** Oracle's `CHAR(n)` blank-pads stored values to length `n`, and Oracle's comparison semantics ignore trailing spaces, so `'ABC' = 'ABC  '` is true. Spark's `STRING` type does no padding and no trailing-space normalization -- the same comparison is false. A join key or lookup that "just worked" against a blank-padded `CHAR` column in Oracle can silently fail to match after a naive `CHAR` &rarr; `STRING` port, dropping rows out of a join with no error raised anywhere.

**`DATE` with a hidden time component.** Oracle's `DATE` type always carries a time-of-day component internally, even for columns everyone in the business calls "just a date" -- `TRUNC(order_date)` is a habit Oracle developers build precisely because the time part is silently there. Spark's `DATE` type has no time component at all; only `TIMESTAMP` does. An LLM that maps Oracle `DATE` &rarr; Spark `DATE` produces code that compiles and looks like the obvious choice, while quietly discarding any time-of-day value that source column actually held -- invisible until someone needs same-day ordering and finds every row from a given day now sorts as a tie.

**NULL-ordering defaults in `ORDER BY`.** Oracle's default puts `NULL`s last in an ascending sort; ANSI SQL's default (and Spark's) puts `NULL`s first. A translated `ORDER BY amount ASC` with no explicit `NULLS LAST` reorders every row with a `NULL` amount to the front of a paginated report instead of the back -- a difference that's invisible in a row-count check and only shows up if someone happens to compare page one of the old report against page one of the new one.

```sql
-- Explicit is the fix for all four: state the Oracle-specific behavior the
-- LLM has no way to infer, rather than trusting the platform default.
ORDER BY amount ASC NULLS LAST
```

{: .important }
> Every one of these four bugs shares the same root cause: the model correctly implements Spark's documented default, and the default is wrong for *this* migration because the source system's default was different. A prompt's Constraints block (from the earlier lecture in this section) is where you close that gap -- name the source system's actual behavior for NULLs, padding, and date semantics explicitly, rather than hoping the model infers it from a schema paste.

None of these four show up as an error anywhere in the pipeline. They show up as a reconciliation mismatch weeks later, or worse, as a number nobody double-checked because the query ran and returned rows. The transaction-semantics gap in the next lecture is a fifth member of this family, sized large enough to deserve its own treatment -- and the audit checklist that closes this section is built specifically to catch all five before any of them reaches production.

<!-- prevnext:start -->

---

| [&larr; Previous: The Window-Function Hallucination]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-window-function-hallucination/' | relative_url }}) | [Next: The Transaction-Semantics Gap &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-transaction-semantics-gap/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

