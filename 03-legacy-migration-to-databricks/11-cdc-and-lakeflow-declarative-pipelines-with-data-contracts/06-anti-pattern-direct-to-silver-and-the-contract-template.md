---
title: "Anti-Pattern: Direct-to-Silver and the Contract Template"
parent: "CDC and Lakeflow Declarative Pipelines With Data Contracts"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/anti-pattern-direct-to-silver-and-the-contract-template/
read_minutes: 4
---

# Anti-Pattern: Direct-to-Silver and the Contract Template
{: .no_toc }

*Estimated read: 4 min*

The incident in the previous lecture had a specific trigger -- a permissive evolution mode plus a missing contract -- but it also had an architectural precondition that made the damage so hard to undo: there was no independent checkpoint between the raw feed and the governed dimension. That precondition has its own name, and it shows up constantly in migrations under schedule pressure: **direct-to-silver**, where a team skips bronze entirely and points `AUTO CDC` straight at a raw source stream to save a layer of pipeline and a bit of storage.

It's tempting because it looks like the medallion architecture minus a redundant hop -- why materialize a raw copy of the data if you're immediately going to transform it into the dimension you actually want? The answer is the same reason a legacy warehouse team never truncated their staging table before validating a load: **bronze is your replay point.** When `AUTO CDC` reads directly from a source stream with no bronze table underneath it, three things are true that weren't true in the layered design from earlier in this section:

- **There's no raw record to reprocess from.** If a bug in the CDC logic corrupts silver, fixing the bug and rerunning means going back to the *source system*, not to a Delta table you already control -- and CDC sources often don't retain more than a few days or hours of replayable history.
- **There's no place to attach a contract.** Expectations attach to a table or view inside the pipeline; skipping bronze means either enforcing the contract on the silver target itself (where a violation now corrupts the audit-history table directly) or not enforcing it at all.
- **Schema drift and business logic fail together.** A malformed row and a legitimate SCD Type 2 update are competing for the same code path with no checkpoint between them, so debugging "why did the dimension go wrong" means untangling ingestion failures from transformation failures in the same trace.

Direct-to-silver isn't wrong because it violates a convention -- it's wrong because it collapses two independent failure domains (ingestion and transformation) into one, and removes the one artifact (a durable bronze table) that made the previous lecture's incident recoverable at all. In the real version of that overnight-rename incident, the fix was rerunning `AUTO CDC` from the retained bronze history after correcting the schema and the contract -- an operation that took an afternoon. Without a bronze table, the same fix means asking the SaaS vendor whether they can replay three days of change events, which they may not be able to do.

The reusable fix is a **contract template** applied at the bronze boundary of every CDC source, adapted per source but structurally consistent:

```python
import dlt

@dlt.table(
    name="<source>_bronze",
    comment="Raw CDC feed, contract-enforced before AUTO CDC consumes it",
)
@dlt.expect_or_fail("business_key_not_null", "<business_key> IS NOT NULL")
@dlt.expect_or_fail("known_operation", "operation IN ('INSERT', 'UPDATE', 'DELETE')")
@dlt.expect_or_fail("sequence_not_null", "<sequence_column> IS NOT NULL")
@dlt.expect_or_drop("no_future_dated_change", "<sequence_column> <= current_timestamp()")
def source_bronze():
    return spark.readStream.table("<source>_raw_autoloader")
```

The three `expect_or_fail` rows are non-negotiable for any CDC source: a null business key, an unrecognized operation code, or a null sequencing value each independently break `AUTO CDC`'s ability to correctly order and key a change, exactly as the phantom-version incident demonstrated. The `expect_or_drop` row is a softer, source-specific guard -- future-dated changes are usually a clock-skew symptom worth quarantining rather than a reason to halt the whole pipeline.

{: .important }
> Treat this template as the minimum bar, not the ceiling. A source with additional business invariants -- an order total that must be non-negative, a status code drawn from a fixed enum -- should add `expect_or_drop` rows for those on top of the three structural checks above. The three structural checks protect `AUTO CDC` itself; everything else protects your specific business logic.

That closes out the pipeline pattern for this section: Auto Loader for ingestion, `AUTO CDC` for the SCD Type 2 target, and a bronze contract standing between them. The quiz that follows checks your grasp of all five preceding lectures together.

<!-- prevnext:start -->

---

| [&larr; Previous: The Schema-Drift Death Spiral]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/the-schema-drift-death-spiral/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/11-cdc-and-lakeflow-declarative-pipelines-with-data-contracts/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

