---
title: "The Physical Design Decision Card"
parent: "Physical Design for Delta: Liquid Clustering Over Indexes"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/the-physical-design-decision-card/
read_minutes: 3
---

# The Physical Design Decision Card
{: .no_toc }

*Estimated read: 3 min*

The previous four lectures cover four separate judgment calls: how Delta's data skipping differs
from indexing, which of the three layout techniques to reach for, how to derive clustering keys from
a workload, and when not to cluster at all. A migration touching hundreds of tables can't afford to
re-run that full discussion for every one of them. The **physical design decision card** compresses
it into a five-question worksheet you fill out once per table, in minutes, from data you already
collected during the workload inventory.

## The card

| Question | What to check | Outcome |
|---|---|---|
| **1. Does a real query filter, join, or group on this table selectively?** | Query-log evidence (Oracle AWR, Teradata DBQL, SQL Server Query Store), not intuition | No &rarr; stop here, leave unclustered |
| **2. Is the table small or fully scanned on every query?** | Row/file count; whether any query reads less than the whole table | Yes &rarr; stop here, leave unclustered |
| **3. Was the table previously partitioned or Z-Ordered?** | Existing `PARTITIONED BY` / `ZORDER BY` definition | Yes &rarr; start the clustering-key shortlist from those columns |
| **4. Is the query pattern stable and well understood, or still evolving?** | Confidence in the workload inventory; whether new reporting is planned | Stable &rarr; manually pin up to 4 keys, highest selectivity first. Evolving &rarr; `CLUSTER BY AUTO` |
| **5. Are any candidate keys highly correlated with each other?** | Column relationships (e.g. `order_date` and `order_month`) | Yes &rarr; drop the coarser column, keep the finer-grained one |

Answering all five takes less time than it took to read this lecture, because every input --
query-log frequency, existing partition definitions, column cardinality -- was already gathered
during the Autopsy and workload-inventory work earlier in this migration. The card doesn't introduce
new analysis; it sequences analysis you've already done into a decision.

## Applying it at estate scale

For a single table, this is a five-minute exercise. For an estate of a few hundred, the value is in
running the same five questions the same way every time, so two different engineers on the migration
team reach the same clustering decision for the same table -- and so a table's clustering choice can
be justified later ("question 3 said start from the existing Z-Order columns") instead of
re-derived from memory.

```sql
-- Outcome of a filled-out card for a fact table previously
-- PARTITIONED BY (region) with periodic OPTIMIZE ... ZORDER BY (customer_id):
ALTER TABLE orders CLUSTER BY (customer_id, order_date);

-- Outcome for a table with a stable existing pattern but planned new
-- reporting workloads not yet in production:
ALTER TABLE inventory_events CLUSTER BY AUTO;
```

{: .important }
> Run the card **before** writing the table's `CREATE TABLE` statement, not after performance
> complaints arrive. Retrofitting Liquid Clustering onto a live table is a cheap `ALTER TABLE`, but
> retrofitting the workload-inventory habit onto a team that's already skipped it for two hundred
> tables is the expensive version of this lecture.

Treat the card as a template to copy into your migration's runbook -- one row per table, five
answers, one outcome -- and the physical-design portion of the migration stops being an open
question by the time you reach the section's quiz.

<!-- prevnext:start -->

---

| [&larr; Previous: Anti-Pattern: Over-Clustering Every Table]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/anti-pattern-over-clustering-every-table/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/06-physical-design-for-delta-liquid-clustering-over-indexes/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

