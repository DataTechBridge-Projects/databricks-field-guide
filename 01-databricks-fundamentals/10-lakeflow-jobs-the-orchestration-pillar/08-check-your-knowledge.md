---
title: "Check Your Knowledge"
parent: "Lakeflow Jobs - The Orchestration Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/check-your-knowledge/
---

# Check Your Knowledge
{: .no_toc }

Test what you picked up across this section -- Lakeflow Jobs' tasks, DAGs, parameterization, task
values, repair runs, and production patterns. This also closes out Part 1 -- Part 2's capstone
project begins next.

1. What does `depends_on` control in a Lakeflow Jobs task definition?
   A. Which cluster the task uses
   B. The dependency relationship that turns a flat list of tasks into an actual DAG, controlling execution order
   C. The task's retry count
   D. The job's schedule

2. How does a notebook task typically receive job parameters?
   A. Through environment variables only
   B. Via `dbutils.widgets`, the same API used for interactive widgets
   C. Through a separate job-parameters API distinct from widgets
   D. Notebook tasks cannot receive parameters

3. How does a Python script task (not a notebook) typically receive parameters?
   A. Via `dbutils.widgets`
   B. As standard command-line arguments (`sys.argv`), commonly parsed with `argparse`
   C. It cannot receive parameters
   D. Through a shared global variable

4. What is a task value used for?
   A. Setting a task's cluster size
   B. Passing a piece of data from one task forward to a downstream task in the same job run
   C. Defining the job's schedule
   D. Storing Unity Catalog credentials

5. What does a Repair run do differently from a full rerun of a failed job?
   A. It reruns every task from scratch
   B. It reruns only the failed task(s) and any downstream-dependent tasks, leaving already-succeeded tasks alone
   C. It only works for single-task jobs
   D. It automatically fixes the underlying bug

6. Why does task idempotency matter specifically for repair runs and backfills?
   A. It doesn't -- idempotency is unrelated to reruns
   B. A repair or backfill reruns a task from the beginning; a non-idempotent task can duplicate work already committed before a failure
   C. Idempotency only matters for streaming tasks
   D. Non-idempotent tasks run faster

7. What is the purpose of `run_if` conditions like `AT_LEAST_ONE_FAILED`?
   A. They control retry counts
   B. They enable conditional branching -- a task only runs if its dependencies' outcomes match the specified condition
   C. They set the job's timezone
   D. They define which cluster a task uses

8. What is a Databricks Asset Bundle used for?
   A. Storing secrets
   B. Deploying a YAML-defined package of jobs, pipelines, and other resources identically across environments via targets
   C. Compressing notebook files
   D. Managing Unity Catalog permissions only

9. Why should production jobs run under a service principal rather than an individual's personal account?
   A. Service principals are faster
   B. A job tied to a personal account breaks if that person's access changes, for reasons unrelated to the job itself
   C. Personal accounts cannot be used with the Jobs API
   D. There is no meaningful difference

10. Why is notifying on every single job run (success and failure alike) discouraged?
    A. It costs extra money per notification
    B. It trains recipients to ignore notifications, so a genuine failure is more likely to go unnoticed
    C. Databricks limits the number of notifications per day
    D. Success notifications are technically impossible

## Answer Key

1. **B** -- `depends_on` establishes the DAG's actual execution dependencies.
2. **B** -- notebook tasks receive parameters through the same `dbutils.widgets` API used interactively.
3. **B** -- Python script tasks receive parameters as command-line arguments.
4. **B** -- task values pass data forward from one task to a downstream task in the same run.
5. **B** -- Repair run reruns only failed/downstream tasks, skipping already-successful ones.
6. **B** -- non-idempotent tasks rerun from the start and can duplicate previously committed work.
7. **B** -- `run_if` conditions enable conditional execution based on dependency outcomes.
8. **B** -- asset bundles deploy the same resource definitions consistently across environments.
9. **B** -- service principals decouple job reliability from any individual's personal account status.
10. **B** -- notification fatigue from routine success emails makes real failures easier to miss.

<!-- prevnext:start -->

---

| [&larr; Previous: Jobs Design Decisions and Production Patterns]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/jobs-design-decisions-and-production-patterns/' | relative_url }}) | [Next: StepRight Capstone Project &rarr;]({{ '/02-stepright-capstone-project/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

