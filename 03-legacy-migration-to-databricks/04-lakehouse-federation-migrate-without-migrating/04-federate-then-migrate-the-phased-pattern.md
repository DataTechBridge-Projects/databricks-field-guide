---
title: "Federate-then-Migrate: The Phased Pattern"
parent: "Lakehouse Federation: Migrate Without Migrating"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/federate-then-migrate-the-phased-pattern/
read_minutes: 3
---

# Federate-then-Migrate: The Phased Pattern
{: .no_toc }

*Estimated read: 3 min*

Put the last two lectures together and a deliberate pattern falls out: federate everything on day
one for governed visibility, then physically migrate workloads in priority order, using federation
as the safety net for whatever hasn't moved yet rather than as a permanent architecture.

**The phased pattern, four stages:**

1. **Federate the entire estate first.** Stand up connections and foreign catalogs for every schema
   in scope, on day one of the technical work, well before any table physically moves. This
   immediately gives every downstream consumer -- dashboards, notebooks, other pipelines -- a single
   governed Unity Catalog surface to point at, instead of some consumers hitting Databricks and
   others still hitting the legacy system directly through separate, ungoverned credentials.
2. **Migrate in the order the 3-R decision and workload inventory already gave you.** Re-architect
   candidates first (highest value, most benefit from the move), rehost candidates last or not at
   all if they're low-value enough to stay federated indefinitely. As each table's physical Delta
   version passes reconciliation (covered in depth later in this part), cut its consumers over from
   the foreign table to the native one.
3. **Let federation cover the gap, table by table.** At any point mid-migration, some tables are
   federated and some are native Delta -- and because both live in the same Unity Catalog namespace
   structure, a consumer querying across both doesn't need to know which is which unless it's writing
   cross-source joins, which is exactly the pattern flagged as risky in the previous lecture.
4. **Decommission federation last, source system last.** Once every table a given legacy schema
   contains has been migrated and reconciled, drop the foreign catalog and, eventually, decommission
   the source system itself -- the last step of the whole migration, not an early one.

```sql
-- Stage 3, example: cut one table over once it's reconciled,
-- leaving the rest of the schema still federated
CREATE OR REPLACE VIEW analytics.sales.orders AS
SELECT * FROM main.sales.orders;  -- now points at migrated Delta table

-- Sibling table not yet migrated continues pointing at the foreign catalog
CREATE OR REPLACE VIEW analytics.sales.returns AS
SELECT * FROM legacy_oracle.sales.returns;  -- still federated
```

Fronting both federated and migrated tables with a stable view layer like this is what lets
downstream consumers stay agnostic to migration progress -- the view's target swaps underneath them
with zero change to any dashboard or notebook query pointed at `analytics.sales.orders`.

{: .important }
> The pattern only works if federation genuinely gets decommissioned as tables migrate -- a federated
> connection that's still live a year after "migration complete" is a sign the phased pattern
> stalled partway through, not a sign federation is a legitimate permanent architecture. Track
> federated-vs-migrated status per schema in the same workload inventory you've been building all
> along, so "how much is left" is always a query away, not a guess.

With Lakehouse Federation established as the bridge -- not the destination -- the next section moves
into the mechanical work of that physical migration itself: translating schema with Lakebridge, and
the blind spots even a good transpiler has.

<!-- prevnext:start -->

---

| [&larr; Previous: The Latency and Cost Penalty: When Federation Backfires]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/the-latency-and-cost-penalty-when-federation-backfires/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/04-lakehouse-federation-migrate-without-migrating/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

