---
title: "The Schema-Drift Death Spiral"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/the-schema-drift-death-spiral/
read_minutes: 3
---

# The Schema-Drift Death Spiral
{: .no_toc }

*Estimated read: 3 min*

Walk through what happens when the two previous lectures' safeguards -- Auto Loader's evolution mode and a bronze-level data contract -- are both set too permissively, using a composite of the kind of incident that shows up in a post-mortem after almost every CDC migration's first quarter in production.

The source system is a third-party order-management SaaS product feeding a CDC stream. Sometime overnight, its vendor ships a routine update that renames `cust_id` to `customer_id` across their outbound event schema -- a change their release notes mention in passing, three paragraphs down, under a heading nobody on the data team subscribes to. The pipeline was configured with `cloudFiles.schemaEvolutionMode` set to `none`, chosen months earlier because an early on-call engineer got tired of restart alerts from the default `addNewColumns` mode and reached for the option that looked like it would "just keep running."

That choice is exactly what turns a routine rename into a silent incident. Under `none`, Auto Loader doesn't fail and doesn't rescue -- it simply drops any column it doesn't recognize. `customer_id` arrives, doesn't match the known `cust_id`, and is discarded before it ever reaches bronze. Every row that night lands with `cust_id` as `NULL`, because the old column name no longer exists in the source payload. Nothing in the ingestion layer raises an alert, because from Auto Loader's point of view, nothing went wrong -- an unrecognized column being dropped is exactly what `none` mode is supposed to do.

Bronze had no `expect_or_fail` on `cust_id IS NOT NULL`, because when the contract was written, `cust_id` being null was considered impossible -- it's the dimension's business key, enforced `NOT NULL` at the source for years. So the null values flow straight through to the `AUTO CDC` flow, which uses `cust_id` as its `keys` column. Here is where the incident compounds instead of staying contained: every incoming row with `cust_id = NULL` is treated by `AUTO CDC` as a *distinct, unmatched key* rather than an update to an existing customer, because `NULL` never equals `NULL` in the merge-key comparison. Instead of updating the current version of each real customer's row, the flow closes out *nothing* and opens a fresh version keyed on null -- and because every null-keyed row looks unique in sequence order, the silver dimension accumulates a new phantom `SCD Type 2` version on every single micro-batch.

By the time someone notices -- typically because a gold-layer customer count report has quietly tripled -- silver contains thousands of orphaned phantom versions, all sharing the same null key, all technically valid rows by the schema the pipeline was enforcing (which is to say: none). Gold aggregates built on top of that silver table are now wrong in a way that isn't a clean "rerun since midnight" fix, because real customer updates from that window were lost, not just duplicated -- the actual `customer_id` values that should have flowed through were dropped at ingestion, so there's no way to reconstruct which phantom row belonged to which real customer without going back to the source system's own change log, if it even retains that far back.

{: .important }
> The root cause here is not any single bad setting -- it's the combination of a *permissive* evolution mode (`none`) with an *absent* contract on the business key. Either one alone would likely have surfaced the problem: `addNewColumns` would have halted the stream on the first unrecognized column, and an `expect_or_fail("cust_id IS NOT NULL")` at bronze would have caught the null flood even with `none` mode still silently dropping the renamed column. A resilient pipeline needs both layers, not one or the other.

The next lecture generalizes this incident into a reusable anti-pattern -- streaming directly into silver without a bronze checkpoint at all -- and gives you a contract template designed to prevent exactly this failure mode.

<!-- prevnext:start -->

---

| [&larr; Previous: Enforcing Schema at Bronze: Data Contracts]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/enforcing-schema-at-bronze-data-contracts/' | relative_url }}) | [Next: Anti-Pattern: Direct-to-Silver and the Contract Template &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/anti-pattern-direct-to-silver-and-the-contract-template/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

