---
title: "Check Your Knowledge"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 7
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test your understanding of this section's AI-assisted migration concepts before moving on.

1. In the hybrid model this section introduces, what does the "20% human accuracy" half actually consist of?
   - A. Rewriting every AI-drafted translation from scratch by hand
   - B. A structured review discipline -- prompts, named failure checks, and a formal audit gate -- applied to AI output
   - C. Waiting for a newer model version to fix the errors automatically
   - D. Disabling AI assistance entirely for procedural code

2. Which of the four required blocks in a deterministic transpilation prompt asks the model to flag its own uncertainty?
   - A. Context
   - B. Contract
   - C. Constraints
   - D. Self-Audit

3. Why does "pinning the model" matter for a transpilation prompt?
   - A. It makes the prompt shorter
   - B. It ensures a rerun of the same prompt later produces a comparable result instead of silent drift from a model update
   - C. It is required by Databricks licensing
   - D. It prevents the model from reading the schema

4. In the window-function hallucination, what specifically goes wrong when an LLM substitutes `DENSE_RANK()` for a source procedure's `RANK()`?
   - A. The query fails to compile
   - B. `DENSE_RANK()`'s gap-free numbering can let a different set of rows pass a `rank <= N` filter than `RANK()` would have
   - C. `DENSE_RANK()` runs but always returns zero rows
   - D. The two functions are fully interchangeable with no behavioral difference

5. What invariant helps detect that `DENSE_RANK()` produced a migrated result, whether or not that was intended?
   - A. The row count always doubles
   - B. The maximum rank value in a partition equals the count of distinct ranked values in that partition
   - C. The minimum rank value is always zero
   - D. Every partition has exactly one row

6. In the "Syntactically Valid, Semantically Wrong" gallery, why does a naive Oracle `DATE` &rarr; Spark `DATE` mapping lose information?
   - A. Spark's `DATE` type stores more precision than Oracle's
   - B. Oracle's `DATE` always carries a time-of-day component that Spark's timeless `DATE` type has no way to hold
   - C. Oracle does not support a `DATE` type
   - D. The mapping causes a compile error, so it's caught immediately

7. Why does an Oracle `CHAR(n)` to Spark `STRING` port risk silently breaking a join?
   - A. `STRING` columns cannot be used in joins
   - B. Oracle blank-pads and ignores trailing spaces in comparisons; Spark's `STRING` does neither, so a value that matched before may not match after
   - C. Oracle `CHAR` columns cannot hold text data
   - D. Spark automatically trims all string columns before joining

8. In the transaction-semantics gap example, what is the correct redesign for an Oracle cursor loop that commits every hundred rows?
   - A. Reproduce a per-hundred-row commit checkpoint literally in PySpark
   - B. Replace the row-by-row commit loop with a single atomic `MERGE INTO` keyed on the record's natural key
   - C. Remove all commit logic and let the table stay in whatever state the job leaves it
   - D. Run the procedure twice and compare results

9. Why must a redesigned transaction boundary (like the one in question 8) be explicitly documented rather than shipped silently?
   - A. Documentation is a compliance requirement unrelated to correctness
   - B. It changes real failure semantics -- from partial-batch commits to all-or-nothing -- which stakeholders need to know about
   - C. Delta Lake requires a comment above every `MERGE` statement
   - D. It has no actual effect on behavior, so documentation is just good practice

10. What does the six-point gate audit consider a "partial pass" -- for example, five of six checks succeeding?
    - A. Acceptable, since most checks passed
    - B. A full fail -- the golden-diff and other checks are binary pass/fail, not partial credit
    - C. Grounds for shipping with a follow-up ticket
    - D. Irrelevant, since only check 1 (row-count parity) actually matters

## Answer Key

1. **B** -- The human-accuracy half is the review discipline itself: structured prompts, named failure-mode checks, and a formal audit gate, not manual rewriting or waiting for a better model.
2. **D** -- The Self-Audit block asks the model to list its own assumptions and flag constructs it's uncertain about.
3. **B** -- Pinning the model version makes a later rerun of the same prompt comparable rather than silently different due to an underlying model update.
4. **B** -- `DENSE_RANK()`'s gap-free numbering can admit a different set of rows through a `rank <= N` filter than `RANK()`'s gapped numbering would, with no error raised.
5. **B** -- `DENSE_RANK()`'s maximum value in a partition always equals the count of distinct ranked values; this equality breaking is the sign `RANK()` (with a tie) was actually intended.
6. **B** -- Oracle's `DATE` type always carries a time-of-day component; Spark's timeless `DATE` type silently discards it on a naive mapping.
7. **B** -- Oracle's `CHAR(n)` blank-pads and ignores trailing spaces in comparisons; Spark's `STRING` does neither, so previously-matching values can fail to match.
8. **B** -- The correct move is a deliberate redesign to a single atomic `MERGE INTO` on the natural key, not a literal translation of the per-row commit loop.
9. **B** -- The redesign changes real failure semantics (partial-batch vs. all-or-nothing), which is a decision stakeholders need to be told about, not discover later.
10. **B** -- The six-point audit treats each check as binary; a golden-diff mismatch or any other failed check is a full fail regardless of how many other checks passed.

<!-- prevnext:start -->

---

| [&larr; Previous: The LLM Prompt Library and Audit Checklist]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-llm-prompt-library-and-audit-checklist/' | relative_url }}) | [Next: The Ingestion Decision Tree &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

