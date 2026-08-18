---
title: "Temp Table to CTE or Lakeflow Declarative Pipeline"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/temp-table-to-cte-or-lakeflow-declarative-pipeline/
read_minutes: 3
---

# Temp Table to CTE or Lakeflow Declarative Pipeline
{: .no_toc }

*Estimated read: 3 min*

A session-scoped temp table exists in legacy procedures for one reason: PL/SQL has no first-class
way to hold an intermediate, tabular result in memory between two steps of a procedure, so a
`CREATE GLOBAL TEMPORARY TABLE` fakes it using real table storage instead. Databricks has two better
answers, and picking the right one depends entirely on one question: **does the intermediate result
get used once, immediately, or does it need to persist and be shared across multiple steps?**

## Used once, immediately: a CTE

If the temp table exists only to stage one intermediate result that the very next statement
consumes and discards, it doesn't need storage at all -- a **common table expression (CTE)** or a
chained PySpark DataFrame does the same job without ever materializing anything:

```sql
-- Legacy: CREATE GLOBAL TEMPORARY TABLE stg_high_value AS SELECT ... ;
--         (next statement reads FROM stg_high_value)
WITH stg_high_value AS (
  SELECT order_id, customer_id, total
  FROM orders
  WHERE total > 1000
)
UPDATE orders SET priority = 'HIGH'
WHERE order_id IN (SELECT order_id FROM stg_high_value);
```

```python
# Equivalent as a DataFrame chain -- no intermediate table, no CTE needed
stg_high_value = orders_df.filter("total > 1000")
orders_df = orders_df.join(stg_high_value.select("order_id"), "order_id", "left_anti") \
                      .unionByName(stg_high_value.withColumn("priority", lit("HIGH")))
```

Both versions compute the intermediate result inline, as part of query planning, rather than writing
it out and reading it back -- which is strictly less work than the legacy pattern ever did, since the
temp table's write was never necessary in the first place.

## Persisted and shared across steps: a Declarative Pipeline flow

If the intermediate result needs to survive across multiple procedure steps, get inspected for
debugging, or feed more than one downstream consumer, collapsing it into a single CTE loses
something real -- visibility into that intermediate state. That's when the intermediate result should
become its own table: a **streaming table or materialized view inside a Lakeflow Declarative
Pipeline**, with its own name, its own lineage, and its own query-ability.

```python
import dlt

@dlt.table(name="stg_high_value_orders")
def stg_high_value_orders():
    return spark.read.table("orders").filter("total > 1000")

@dlt.table(name="orders_flagged")
def orders_flagged():
    stg = dlt.read("stg_high_value_orders")
    return stg.withColumn("priority", lit("HIGH"))
```

Because the pipeline's DAG shows `stg_high_value_orders` as a named node feeding
`orders_flagged`, anyone debugging the pipeline can query the intermediate table directly --
something a session-scoped temp table never allowed, since it vanished with the session that
created it.

## The decision in one line

| Signal | Choose |
|---|---|
| Used once, by the very next statement, never inspected independently | CTE or a chained DataFrame |
| Reused by multiple later steps, or valuable to inspect on its own | A named table in a Declarative Pipeline |

{: .important }
> Don't default to materializing every temp table as its own persisted table just because that's
> what the legacy code did -- most temp tables in real procedures are single-use staging that a CTE
> handles for free. Reserve the Declarative Pipeline table for the cases where the intermediate
> result genuinely earns its own name and lineage.

The next lecture covers the pattern nearly every one of these procedures eventually needs regardless
of which path this decision takes: the upsert, and its direct translation from Oracle `MERGE` to
Delta's `MERGE INTO`.

<!-- prevnext:start -->

---

| [&larr; Previous: Trigger to CDC, DBMS_SCHEDULER to Workflows]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/trigger-to-cdc-dbms-scheduler-to-workflows/' | relative_url }}) | [Next: Oracle MERGE to Delta Lake MERGE INTO &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/oracle-merge-to-delta-lake-merge-into/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

