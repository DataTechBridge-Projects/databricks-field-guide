---
title: "Writing Effective PL/SQL to PySpark Transpilation Prompts"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/writing-effective-pl-sql-to-pyspark-transpilation-prompts/
read_minutes: 3
---

# Writing Effective PL/SQL to PySpark Transpilation Prompts
{: .no_toc }

*Estimated read: 3 min*

An LLM answers the prompt you actually wrote, not the one you meant to write, and "convert this PL/SQL to PySpark" is an underspecified prompt that invites the model to fill every gap with its own default assumption -- usually the syntactically nearest one, not the semantically correct one. **A deterministic transpilation prompt has four required blocks: Context, Contract, Constraints, and Self-Audit.** Skipping any one of them is where the failure modes in the next three lectures come from.

**Context** is the raw material the model needs and none of it should be paraphrased. Paste the actual `CREATE TABLE` DDL for every table the procedure touches, including nullability and precision -- not "a customer table with an ID and a balance," but the real `NUMBER(12,2)` and `NOT NULL` constraints. Paste the full procedure body, not a summary of what it does. If a data-quality workaround or a business rule isn't obvious from the code, state it in one sentence; the model can't recover intent the code doesn't express.

**Contract** states precisely what comes back: "Return a single PySpark function that takes DataFrames as input and returns a DataFrame -- no prose explanation, no partial snippets, and preserve every source column name and type exactly unless a stated constraint requires a change."

**Constraints** are the rules that rule out the model's convenient defaults: "translate the cursor loop to a set-based DataFrame operation, not a `.collect()` and Python loop," "use `MERGE INTO` for any upsert pattern instead of separate `UPDATE`/`INSERT` branches," "preserve the source's `RANK()` vs `DENSE_RANK()` choice exactly -- do not substitute one for the other," "match Oracle's NULL-handling semantics, not Spark's ANSI defaults, unless told otherwise."

**Self-Audit** asks the model to grade its own uncertainty before you do: "List every assumption you made where the source code was ambiguous, and flag any construct you're not fully confident has an exact PySpark equivalent." This block routinely surfaces the exact spots worth double-checking by hand -- a model that says "I assumed `NULLS LAST` ordering since none was specified" has just handed you a reconciliation test to run.

```text
CONTEXT: [DDL for every touched table] + [full procedure source] + [one-line business intent, if not obvious]
CONTRACT: [exact output format -- function signature, DataFrame in/out, no prose]
CONSTRAINTS: [set-based, not row-by-row] + [MERGE INTO, not UPDATE/INSERT pairs]
            + [preserve RANK vs DENSE_RANK exactly] + [Oracle NULL semantics, not Spark defaults]
SELF-AUDIT: "List every assumption and every construct you're uncertain about."
```

Two habits make this repeatable rather than a one-off trick. First, **pin the model** -- record the exact model name and version used for a given migration wave, the same way you'd pin a compiler version for a build, so a rerun six weeks later produces a comparable result instead of a silently different one from a model that's since been updated. Second, keep the underlying schema paste current; a prompt built against last month's DDL export will confidently mistranslate a column that's since changed type. Some teams run this same four-block prompt through an in-workspace assistant like Databricks' **[Genie Code](https://docs.databricks.com/aws/en/notebooks/databricks-assistant-faq)** so the draft lands next to the Unity Catalog schema it's reasoning about; others use an external model and paste the schema in by hand. Either way, the prompt library in this section's final lecture is what makes that pinned, schema-current prompt reusable instead of retyped from memory every time.

<!-- prevnext:start -->

---

| [&larr; Previous: The Hybrid Model: 80% AI Speed plus 20% Human Accuracy]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-hybrid-model-80-ai-speed-plus-20-human-accuracy/' | relative_url }}) | [Next: The Window-Function Hallucination &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-window-function-hallucination/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

