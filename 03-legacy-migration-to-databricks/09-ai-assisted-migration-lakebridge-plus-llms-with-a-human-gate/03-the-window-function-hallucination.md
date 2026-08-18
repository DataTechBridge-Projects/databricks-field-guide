---
title: "The Window-Function Hallucination"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-window-function-hallucination/
read_minutes: 3
---

# The Window-Function Hallucination
{: .no_toc }

*Estimated read: 3 min*

Here's a translation that runs without error, returns a plausible-looking result, and is quietly wrong. An Oracle procedure ranks each customer's orders by amount using **`RANK()`**, and a downstream report filters to `WHERE order_rank <= 3` to show each customer's top three orders. Ask an LLM to "convert this ranking logic to a PySpark window function" without pinning down which function, and it will frequently return **`dense_rank()`** instead -- not because it misread the code, but because `dense_rank()` shows up far more often in the generated PySpark examples the model has seen, and nothing in an underspecified prompt told it the choice mattered.

```sql
-- Source: Oracle
RANK() OVER (PARTITION BY customer_id ORDER BY order_amount DESC) AS order_rank
```

```python
# LLM draft: silently swapped
from pyspark.sql import functions as F, Window
w = Window.partitionBy("customer_id").orderBy(F.col("order_amount").desc())
df.withColumn("order_rank", F.dense_rank().over(w))
```

The two functions agree everywhere there are no ties and disagree everywhere there are. `RANK()` assigns `1, 1, 3, 4` to a tie at the top followed by two more rows -- it leaves a gap where the tied rank would have consumed a slot. `DENSE_RANK()` assigns `1, 1, 2, 3` for the same data -- no gaps, ever. Feed either sequence into `WHERE order_rank <= 3` and you get a *different set of rows* the moment a tie occurs anywhere above the cutoff: `DENSE_RANK()`'s compressed numbering lets a fourth distinct order amount into "top 3" that `RANK()` would have excluded, because `RANK()`'s gap already used up that slot on the tie. No error, no warning -- just a top-3 report that includes one more or one fewer order than the source system ever did, for every customer who happened to have a tie.

**The invariant that catches this before it ships: for any partition, `DENSE_RANK()`'s maximum value always equals the count of distinct values being ranked in that partition; `RANK()`'s maximum value equals that same distinct count only when there happen to be zero ties.** Run this as a validation query against the migrated output:

```sql
SELECT customer_id,
       MAX(order_rank)                       AS max_rank,
       COUNT(DISTINCT order_amount)          AS distinct_amounts
FROM   migrated_orders
GROUP BY customer_id
HAVING MAX(order_rank) = COUNT(DISTINCT order_amount)
   AND EXISTS (SELECT 1 FROM migrated_orders m2
               WHERE m2.customer_id = migrated_orders.customer_id
               GROUP BY m2.order_amount HAVING COUNT(*) > 1)
```

Any customer that shows up in that result has a tie in their order amounts *and* a rank sequence with no gaps -- proof that `DENSE_RANK()` produced the output, whether or not that's what the source procedure actually specified. Cross-reference against the original PL/SQL: if it said `RANK()`, you've caught the hallucination before the report ships.

{: .important }
> `RANK()` and `DENSE_RANK()` are the sharpest example, but the same silent-substitution risk applies to any pair of near-synonym functions an LLM has seen used interchangeably in training data. Whenever a prompt's Constraints block doesn't name the exact function, assume the model will pick whichever one it has seen more often -- not whichever one the source actually used. See [`RANK`](https://docs.databricks.com/aws/en/sql/language-manual/functions/rank) and [`DENSE_RANK`](https://docs.databricks.com/aws/en/sql/language-manual/functions/dense_rank) in the SQL language reference for the exact tie-handling rules.

This is one instance of a broader category of bug -- code that compiles, runs, and returns numbers that look reasonable while being subtly wrong. The next lecture catalogs several more of these before the section turns to fixing the process that produces them.

<!-- prevnext:start -->

---

| [&larr; Previous: Writing Effective PL/SQL to PySpark Transpilation Prompts]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/writing-effective-pl-sql-to-pyspark-transpilation-prompts/' | relative_url }}) | [Next: Syntactically Valid, Semantically Wrong &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/syntactically-valid-semantically-wrong/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

