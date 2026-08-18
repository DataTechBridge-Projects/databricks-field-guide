---
title: "The Migration Assessment Scorecard"
parent: "The 3-R Decision and the TCO That Convinces the CFO"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/the-migration-assessment-scorecard/
read_minutes: 3
---

# The Migration Assessment Scorecard
{: .no_toc }

*Estimated read: 3 min*

The decision tree in the previous lecture is the right *mental model*, but "is business value high?"
is a judgment call two people on the same team can disagree about. The **migration assessment
scorecard** turns that judgment into six numeric dimensions, each rated 1-5 per workload, so the 3-R
call is a reproducible score rather than whoever argued more persuasively in the meeting:

| Dimension | 1 (low) | 5 (high) | Source |
|---|---|---|---|
| **Technical complexity** | Simple `SELECT`/`INSERT` logic | Deep nesting, dynamic SQL, recursive calls | Dependency graph (previous section) |
| **PL/SQL depth** | Little to no procedural code | Heavy cursor loops, package-level state, exception handling chains | Procedure line count, cyclomatic complexity |
| **Business value** | Rarely referenced, no downstream reporting | Feeds board-level metrics or regulatory reporting | Stakeholder interviews, downstream consumer count |
| **Change frequency** | Untouched for 2+ years | Modified multiple times per quarter | Version control / change-ticket history |
| **Data volume** | Under 10 GB | Multi-terabyte, high-growth | Workload inventory `storage_gb` |
| **Dependency fanout** | Zero or one dependent object | Hub procedure with dozens of dependents | Workload inventory `dependent_object_count` |

Score every workload from the inventory across all six, and two aggregate views fall out:

```python
# workload_score = average of the six 1-5 dimension scores
# re_architect_score = business_value + change_frequency + pl_sql_depth  (max 15)
# rehost_score = 15 - re_architect_score  (inverse signal)

df = df.withColumn(
    "re_architect_score",
    F.col("business_value") + F.col("change_frequency") + F.col("pl_sql_depth")
).withColumn(
    "recommended_strategy",
    F.when(F.col("re_architect_score") >= 11, "Re-architect")
     .when(F.col("re_architect_score") >= 7, "Re-platform")
     .otherwise("Rehost")
)
```

Weighting business value, change frequency, and PL/SQL depth together (rather than treating all six
dimensions equally) reflects the decision tree's actual logic from the previous lecture: complexity
and data volume matter for *effort estimation*, but value and change frequency are what determine
*whether the re-architecture investment pays off*. A workload can be technically complex and still
be a poor re-architect candidate if nobody asks it to change.

{: .important }
> The scorecard is a starting recommendation, not a binding verdict -- an architect can and should
> override a borderline score with domain knowledge the six dimensions don't capture (an upcoming
> regulatory deadline, a planned deprecation). What it prevents is the more common failure: an
> entirely undocumented, unreproducible 3-R call that nobody can defend six months later when
> someone asks why workload X got re-architected and workload Y, which looks similar, didn't.

Scoring answers "what strategy." It doesn't answer "is this worth the money." That's the next
lecture's job: turning the scorecard's output into a three-year total cost of ownership model.

<!-- prevnext:start -->

---

| [&larr; Previous: Rehost vs Re-platform vs Re-architect: The Decision Tree]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/rehost-vs-re-platform-vs-re-architect-the-decision-tree/' | relative_url }}) | [Next: Building the 3-Year TCO Calculator &rarr;]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/building-the-3-year-tco-calculator/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

