---
title: "Cursor to Set-Based DataFrame Ops"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/cursor-to-set-based-dataframe-ops/
read_minutes: 3
---

# Cursor to Set-Based DataFrame Ops
{: .no_toc }

*Estimated read: 3 min*

The Autopsy section established the principle: a cursor loop is mechanism, not business logic, and
the statement inside it is what actually needs to migrate. This lecture is where that principle
becomes working code -- the concrete techniques for turning row-at-a-time cursor processing into a
single set-based operation that Spark's distributed engine can actually parallelize.

## The three shapes a cursor body takes

Almost every cursor loop found in legacy procedures reduces to one of three patterns, each with a
direct set-based replacement:

**1. Conditional update per row.** A cursor that fetches rows and conditionally updates a target
table based on some check becomes a single `UPDATE ... WHERE` or `MERGE INTO`, with the cursor's
condition folded into the `WHERE` clause:

```sql
-- Legacy: FOR r IN (SELECT * FROM stg_orders) LOOP
--           IF r.total > 1000 THEN UPDATE orders SET priority = 'HIGH' WHERE order_id = r.order_id; END IF;
--         END LOOP;
UPDATE orders SET priority = 'HIGH'
WHERE order_id IN (SELECT order_id FROM stg_orders WHERE total > 1000);
```

**2. Running accumulation.** A cursor that walks rows in order, maintaining a running total or
counter (`v_running_total := v_running_total + r.amount`), maps to a **window function** --
`SUM(...) OVER (ORDER BY ...)` -- rather than to any kind of loop:

```python
from pyspark.sql import Window
from pyspark.sql.functions import sum as _sum

running_total = Window.partitionBy("customer_id").orderBy("order_date")
df.withColumn("running_total", _sum("amount").over(running_total))
```

**3. Per-row external call.** A cursor that calls out to something genuinely row-scoped -- an
external API, a stored function with no set-based equivalent -- is the one case where a literal loop
survives, expressed as a PySpark **UDF** applied across the DataFrame rather than a Python `for`
loop calling `.collect()` first. Reach for this only after confirming the first two patterns don't
apply; it's the fallback, not the default.

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

@udf(returnType=StringType())
def classify_via_external_rule(amount):
    return "HIGH" if amount > 1000 else "STANDARD"

df.withColumn("priority", classify_via_external_rule("amount"))
```

## Why the distinction matters at scale

A cursor processing ten thousand rows sequentially on a single session is slow but bounded --
Oracle runs it to completion on one thread. The same pattern ported literally into PySpark as a
`for row in df.collect(): ...` loop is worse in a distributed engine, not better: `collect()` pulls
every row back to the driver, discarding the cluster's parallelism entirely and risking a driver
out-of-memory error on anything beyond a small dataset. The set-based rewrite isn't a style
preference -- it's the difference between using Spark's distributed engine and defeating it.

{: .important }
> Before reaching for a UDF, check whether the cursor's logic is really row-scoped or whether it's
> actually a window function, a `groupBy().agg()`, or a plain `WHERE` clause in disguise. Most
> cursor loops found in real legacy code turn out to be pattern 1 or 2 -- the genuinely row-scoped
> pattern 3 is the exception, not the common case.

The next lecture applies the same "find the set-based equivalent" discipline to a harder case:
triggers, where the row-level logic isn't even called explicitly -- it fires as a side effect of a
write.

<!-- prevnext:start -->

---

| [&larr; Previous: Pattern Translation: Cursors, Triggers, Temp Tables, MERGE]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/' | relative_url }}) | [Next: Trigger to CDC, DBMS_SCHEDULER to Workflows &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/trigger-to-cdc-dbms-scheduler-to-workflows/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

