---
title: "Anti-Pattern Gallery and Pattern Cheat Sheet"
parent: "Pattern Translation: Cursors, Triggers, Temp Tables, MERGE"
grand_parent: "Legacy Migration to Databricks"
nav_order: 6
permalink: /03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/anti-pattern-gallery-and-pattern-cheat-sheet/
read_minutes: 3
---

# Anti-Pattern Gallery and Pattern Cheat Sheet
{: .no_toc }

*Estimated read: 3 min*

Every pattern in this section has a literal, line-by-line translation that compiles, runs, and
quietly performs badly or produces subtly wrong results. This gallery catalogs the five translations
experienced migration teams learn to recognize and avoid, followed by a one-page cheat sheet for the
whole section.

## The gallery

**1. The `collect()` loop.** A cursor translated as `for row in df.collect(): ...` instead of a
set-based operation. It runs, but pulls the entire DataFrame to the driver first, discarding
distributed execution and risking an out-of-memory failure on any dataset larger than the driver's
memory.

```python
# Anti-pattern
for row in df.collect():
    if row.total > 1000:
        update_priority(row.order_id, "HIGH")

# Correct
df_high = df.filter("total > 1000")
target_df.join(df_high.select("order_id"), "order_id") \
         .withColumn("priority", lit("HIGH"))
```

**2. The polling job pretending to be a trigger.** Instead of consuming Change Data Feed, a Lakeflow
Job runs every five minutes and does a full-table comparison to detect what changed -- reinventing
CDC badly, at far more compute cost than reading `table_changes()` directly.

**3. The unconditional temp-table materialization.** Every intermediate result gets written out as a
persisted Declarative Pipeline table "to be safe," even single-use staging that a CTE would compute
for free. This inflates both storage and pipeline DAG complexity with nodes nobody ever queries.

**4. The `MERGE` with an ambiguous join key.** A `MERGE INTO` whose `ON` condition doesn't uniquely
identify target rows, ported directly from a legacy `MERGE` that happened to tolerate duplicate
matches. Delta's stricter behavior surfaces this as a runtime error -- treat that error as a
data-quality finding to fix, not an obstacle to suppress.

**5. The hand-rolled SCD Type 2 that assumes ordered arrival.** A two-pass `MERGE` implementation of
SCD Type 2 that silently assumes updates arrive in timestamp order, breaking the moment a late or
out-of-order record shows up -- exactly the case `create_auto_cdc_flow`'s `sequence_by` parameter
exists to handle correctly.

{: .important }
> The common thread across all five: each one is a *literal* translation that preserves the legacy
> code's shape instead of its intent. Every anti-pattern in this gallery still "works" in a demo with
> a small sample dataset -- they fail at production data volume or under real-world arrival order,
> which is exactly why they survive code review and surface later as performance or correctness
> incidents.

## The pattern cheat sheet

| Legacy construct | Symptom if translated literally | Databricks-native pattern |
|---|---|---|
| Cursor loop | Row-at-a-time processing defeats distributed execution | Set-based DataFrame/SQL op, window function, or UDF only for genuinely row-scoped logic |
| Row-level trigger | No direct equivalent; hidden, unqueryable execution | Delta Change Data Feed (`table_changes()`, `readChangeFeed`) |
| `DBMS_SCHEDULER` job | Scheduling/retry logic buried in scheduler catalog | Lakeflow Jobs task with declared schedule, retries, dependencies |
| Session temp table | Materializing single-use staging as a real table | CTE or chained DataFrame (single-use); Declarative Pipeline table (reused/shared) |
| Oracle `MERGE` | Ambiguous join keys silently tolerated | Delta `MERGE INTO`, which rejects ambiguous matches by default |
| SCD Type 2 via cursor + two DML statements | Breaks silently on out-of-order arrivals | `create_auto_cdc_flow` with `stored_as_scd_type=2` |

Keep this table next to the decomposition worksheet from the previous section -- together they cover
every "target pattern" entry a real procedure autopsy is likely to produce. The section's quiz next
checks recall of both the individual patterns and the reasoning behind each anti-pattern.

<!-- prevnext:start -->

---

| [&larr; Previous: SCD Type 2 the Lakehouse Way]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/scd-type-2-the-lakehouse-way/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/08-pattern-translation-cursors-triggers-temp-tables-merge/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

