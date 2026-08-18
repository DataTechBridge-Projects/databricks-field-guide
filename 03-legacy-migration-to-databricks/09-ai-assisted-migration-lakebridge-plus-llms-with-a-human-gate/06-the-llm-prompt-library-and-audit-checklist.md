---
title: "The LLM Prompt Library and Audit Checklist"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-llm-prompt-library-and-audit-checklist/
read_minutes: 3
---

# The LLM Prompt Library and Audit Checklist
{: .no_toc }

*Estimated read: 3 min*

A four-block prompt that lives only in one engineer's chat history is a one-off. The same prompt, versioned in git alongside the procedure it translates, is an asset the whole migration team can reuse, re-run, and audit. **Store each procedure's transpilation prompt as a YAML file, one per source object, checked into the same repository as the migration code itself.**

```yaml
id: proc_customer_order_rank_v1
source_object: SALES.PKG_ORDERS.RANK_CUSTOMER_ORDERS
model_pinned: <model-name>-<version>            # reproducibility, not just documentation
context:
  ddl_ref: schemas/sales_orders.sql
  procedure_ref: source/pkg_orders_rank.sql
  business_note: "Ties on order_amount are intentional -- RANK(), not DENSE_RANK()."
contract: "Return one PySpark function, DataFrame in, DataFrame out. No prose."
constraints:
  - set_based_only: true
  - upsert_pattern: MERGE_INTO
  - ranking_function: RANK        # names the exact function -- see lecture 3
  - null_ordering: ORACLE_DEFAULT # NULLS LAST on ASC
last_validated_commit: a1b2c3d
audit_status: passed
```

Pinning the model name and version in the same file as the prompt means a re-run of this exact procedure eighteen months from now, against a newer model, is a deliberate, comparable decision -- not a silent drift. The `audit_status` and `last_validated_commit` fields turn the library into a running record of which translations have actually cleared review, not just which ones exist.

That review is the **six-point gate audit** every AI-drafted translation runs through before merge:

1. **Row-count parity.** Migrated output row count matches a source extract for the same key range and date window, exactly.
2. **Golden-diff match.** The migrated output is diffed, row for row and column for column, against a preserved baseline -- either the source system's real output or the first hand-verified translation of this procedure. **Zero missing rows and zero extra rows** is the pass bar; a partial match is a fail, not a partial pass.
3. **Checksum or hash reconciliation** on the numeric and monetary columns, catching the rounding and type-coercion drift a row-count check alone would miss.
4. **Named-invariant check** for any window function, ranking, or ordering logic -- the `max(rank) = distinct count` test from the window-function lecture, run as a standing query, not a one-time glance.
5. **NULL and date-semantics spot check** against the empty-string, blank-padding, and `DATE`-versus-`TIMESTAMP` gotchas from the syntactically-wrong gallery.
6. **Transaction-boundary sign-off.** If atomicity was redesigned (a `MERGE INTO` replacing a row-by-row commit loop, for instance), a named reviewer confirms that redesign was reviewed and documented, not silently introduced.

{: .important }
> A translation that passes five of six checks is not "mostly done" -- it's failed. Golden-diff parity in particular is binary: a single missing or duplicated row means something in the upsert logic or the join cardinality is wrong, and it needs to be found before this procedure ships, not tracked as a known issue.

This checklist is what "20% human accuracy" from the first lecture actually means when it's operationalized: not one senior architect's careful judgment applied inconsistently across forty procedures, but a fixed six-item gate that any reviewer on the team runs the same way every time. The reconciliation techniques behind checks 2 and 3 -- count, sum, checksum, and hash comparisons at scale -- get their own dedicated treatment in the Reconciliation Stack section later in this part; here, they're the audit gate that decides whether an AI-drafted translation is ready to leave this section at all.

<!-- prevnext:start -->

---

| [&larr; Previous: The Transaction-Semantics Gap]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-transaction-semantics-gap/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

