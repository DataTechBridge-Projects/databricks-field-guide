---
title: "The Hybrid Model: 80% AI Speed plus 20% Human Accuracy"
parent: "AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/the-hybrid-model-80-ai-speed-plus-20-human-accuracy/
read_minutes: 3
---

# The Hybrid Model: 80% AI Speed plus 20% Human Accuracy
{: .no_toc }

*Estimated read: 3 min*

Back in [The 80/20 Truth]({{ '/03-legacy-migration-to-databricks/01-why-enterprise-migrations-fail-and-the-architect-who-stops-it/the-80-20-truth-what-lakebridge-does-and-the-20-it-never-will/' | relative_url }}), the 80/20 split was Lakebridge's own: its transpiler engines handle the mechanical 80% of DDL and standard SQL, and a human handles the 20% that requires judgment. This section is about a narrower, sharper version of that same split, one that shows up *inside* the 20% -- the procedural logic your Autopsy worksheet flagged as too tangled for Lakebridge's rule-based converter to touch at all. That's where teams increasingly point a general-purpose **large language model (LLM)** directly at a stored procedure and ask for a PySpark draft. It works, and it's fast. It is also the point in a migration where "the AI wrote it" becomes the most dangerous sentence in the project, because the output is a plausible guess dressed up as a translation, not a verified fact.

**The hybrid model treats an LLM's output the same way you'd treat a first-pass translation from a smart but unfamiliar junior contractor: fast, often right, and never trusted without a review step.** Feed a well-structured prompt forty lines of Oracle PL/SQL -- a cursor loop, a couple of conditional branches, an embedded `MERGE` -- and a competent model will return working PySpark in seconds. Doing the same conversion by hand, reading the procedure line by line and reasoning through every branch, might take a migration engineer half a day per procedure. Multiply that across a workload inventory of forty or fifty procedures and the AI-assisted path can compress weeks of translation work into a couple of days. That's the **80% AI speed** half of this lecture's title, and it's real.

The **20% human accuracy** half is the discipline that keeps that speed from becoming a liability. An LLM has no access to your reconciliation results, no memory of the data-quality workaround buried in a `WHERE` clause from 2014, and no way to know whether the column it just cast from `DATE` to `date` silently dropped a time-of-day value someone downstream depends on. It optimizes for "output that looks like a correct translation," which is a different objective than "output that *is* a correct translation" -- and the gap between those two objectives is exactly where the next few lectures live: a window-function swap that silently drops rows, NULL and date-arithmetic mismatches that pass every syntax check, and a transaction-boundary redesign that has to be a deliberate call, not an accident.

{: .important }
> A stored procedure that fails to compile gets caught immediately, by anyone. A stored procedure that compiles, runs, and returns a *plausible but wrong* number is the failure mode that ships to production and surfaces three weeks later as a finance team asking why the quarterly total doesn't match. Treat AI-drafted translations as guilty until reconciled, not innocent until proven wrong.

The rest of this section builds the operating model for that review step: how to write a prompt precise enough to reduce the model's room to guess (next lecture), three concrete failure patterns worth knowing by name before you see them in your own output, and a versioned prompt library with a formal audit gate that turns "one careful architect reviewing everything" into a checklist the whole team can run consistently on the next wave of procedures.

<!-- prevnext:start -->

---

| [&larr; Previous: AI-Assisted Migration: Lakebridge plus LLMs With a Human Gate]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/' | relative_url }}) | [Next: Writing Effective PL/SQL to PySpark Transpilation Prompts &rarr;]({{ '/03-legacy-migration-to-databricks/09-ai-assisted-migration-lakebridge-plus-llms-with-a-human-gate/writing-effective-pl-sql-to-pyspark-transpilation-prompts/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

