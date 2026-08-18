---
title: "The Transaction-Semantics Gap"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-transaction-semantics-gap/
read_minutes: 3
---

# The Transaction-Semantics Gap
{: .no_toc }

*Estimated read: 3 min*

Consider an Oracle procedure that processes an incoming batch through a cursor: for each row, it either `UPDATE`s a matching record or `INSERT`s a new one, and it issues a `COMMIT` every hundred rows to keep the redo log manageable. If the procedure dies at row 8,347, rows 1 through 8,300 are already durably committed -- the failure leaves the target table in a well-defined partial state, and Oracle's per-statement, per-batch commit boundary is *part of the procedure's design*, not an incidental detail.

Ask an LLM to translate this "faithfully," and it will often do exactly that: emit a loop that performs a row-by-row `UPDATE`-or-`INSERT` with something resembling a commit checkpoint every hundred rows, because that's the most literal reading of the source. This is doubly wrong. It's slow, for the row-at-a-time reason covered back in the Procedure Autopsy section -- Spark's engine is built for set-based operations, not per-row loops. But it's also **semantically hollow**, because Delta Lake has no concept of a mid-job commit checkpoint the way Oracle does. A Delta write is atomic per table operation -- a `MERGE`, an `UPDATE` -- and a Spark job that dies partway through a hundred-row Python loop doesn't leave a clean "first 8,300 rows are safely committed" boundary behind; it leaves whatever partial, non-transactional mess a failed Python loop happens to leave, which is a *worse* failure mode than the one it was trying to reproduce.

**The correct move is not a more faithful translation -- it's a deliberate redesign: replace the row-by-row commit loop with a single atomic `MERGE INTO` keyed on the record's natural key.**

```sql
MERGE INTO target_orders AS t
USING incoming_batch AS s
ON    t.order_id = s.order_id          -- natural key, not a surrogate row number
WHEN MATCHED THEN
  UPDATE SET t.status = s.status, t.amount = s.amount, t.updated_at = s.updated_at
WHEN NOT MATCHED THEN
  INSERT (order_id, status, amount, updated_at)
  VALUES (s.order_id, s.status, s.amount, s.updated_at)
```

This single **[`MERGE INTO`](https://docs.databricks.com/aws/en/delta/merge)** either applies to the entire batch or, on failure, applies to none of it -- Delta's transaction log guarantees that atomicity. The failure semantics genuinely change: instead of "the first 8,300 of 10,000 rows are committed," a failed job now means "zero of 10,000 rows are committed, retry the whole batch." That's a real behavioral difference from the source system, and it needs to be surfaced to the business owner and the data-quality team as an explicit design decision -- "we're moving from partial-batch commits to all-or-nothing batch commits, here's why, and here's what it means for retry behavior" -- not discovered by an auditor three months later comparing failure logs.

{: .important }
> When a source procedure's transaction boundaries don't map onto the target platform, the right answer is never "translate it as literally as possible anyway." Redesign the atomicity boundary deliberately, on a natural key, and document the change -- an LLM asked to be "faithful" to the source will default to the literal, usually wrong, choice unless a prompt's Constraints block tells it otherwise.

This is the last of the section's three named failure patterns. The closing lecture turns all three -- the window-function swap, the syntactically-valid-but-wrong gallery, and this transaction gap -- into a checklist an entire migration team can run against every AI-drafted translation, not just the ones one careful reviewer happens to catch.

<!-- prevnext:start -->

---

| [&larr; Previous: Syntactically Valid, Semantically Wrong]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/syntactically-valid-semantically-wrong/' | relative_url }}) | [Next: The LLM Prompt Library and Audit Checklist &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-llm-prompt-library-and-audit-checklist/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

