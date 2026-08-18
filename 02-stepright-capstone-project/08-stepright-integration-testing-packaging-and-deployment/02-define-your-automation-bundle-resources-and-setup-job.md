---
title: "Define your Automation Bundle Resources and Setup Job"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 2
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/define-your-automation-bundle-resources-and-setup-job/
read_minutes: 13
---

# Define your Automation Bundle Resources and Setup Job
{: .no_toc }

*Estimated read: 13 min*

Lecture 1 previewed the pattern with one pipeline. This lecture rewrites all four resource files in
full -- bronze pipeline, silver-gold pipeline, orchestration job, dashboard -- and adds one new
resource none of the previous seven sections needed: an environment bootstrap job that creates the
catalog, schema, and volumes a brand-new target starts without.

## `resources/pipeline_bronze.yml`

```yaml
resources:
  pipelines:
    bronze_cdc:
      name: steprightproject-bronze-cdc
      catalog: ${var.catalog}
      schema: ${var.schema}
      serverless: true
      continuous: false
      libraries:
        - glob:
            include: transformations/bronze_*.py
```

## `resources/pipeline_silver_gold.yml`

```yaml
resources:
  pipelines:
    silver_gold:
      name: steprightproject-silver-gold
      catalog: ${var.catalog}
      schema: ${var.schema}
      serverless: true
      continuous: false
      libraries:
        - glob:
            include:
              - transformations/silver_*.py
              - transformations/gold_*.py
              - transformations/gold_*.sql
```

Both pipeline resources carry the exact glob scoping [Section 5, Lecture 1]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/design-the-orchestration-and-job-flow/' | relative_url }})
designed when it split the original combined pipeline in two -- nothing about that design changes
here, only where the definition lives and how it picks up `catalog`/`schema` per target.

## `resources/orchestration_job.yml`, with real cross-resource references

```yaml
resources:
  jobs:
    stepright_daily_pipeline:
      name: "StepRight Daily Pipeline"
      schedule:
        quartz_cron_expression: "0 0 3 * * ?"
        timezone_id: "America/New_York"
      email_notifications:
        on_failure: ["${var.notification_email}"]
      parameters:
        - name: run_date
          default: "{{job.trigger.time.iso_date}}"
      tasks:
        - task_key: run_ingestion
          pipeline_task:
            pipeline_id: ${resources.pipelines.bronze_cdc.id}

        - task_key: dq_check
          depends_on: [{task_key: run_ingestion}]
          python_wheel_task:
            package_name: "steprightproject"
            entry_point: "dq_check"
            parameters: ["--run_date", "{{job.parameters.run_date}}"]

        - task_key: run_transformation
          depends_on: [{task_key: dq_check}]
          run_if: ALL_SUCCESS
          pipeline_task:
            pipeline_id: ${resources.pipelines.silver_gold.id}

        - task_key: report
          depends_on: [{task_key: run_transformation}]
          python_wheel_task:
            package_name: "steprightproject"
            entry_point: "report"
            parameters: ["--run_date", "{{job.parameters.run_date}}"]
```

The only real change from [Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})'s
version: `${var.bronze_pipeline_id}` and `${var.silver_gold_pipeline_id}` become
`${resources.pipelines.bronze_cdc.id}` and `${resources.pipelines.silver_gold.id}` now that both
pipelines are bundle-managed resources in the same `databricks.yml`, sitting right alongside this
job -- a direct reference beats a variable once the thing being referenced lives in the same
bundle, since the bundle framework resolves it automatically rather than requiring a human to keep
a variable's value in sync with whatever pipeline ID actually got created.

## Testing bootstrap idempotency, deliberately, before trusting it in CI

Before Lecture 6 ever calls `environment_bootstrap` automatically, run it twice by hand against
`dev` and confirm the second run produces no errors and no duplicate objects:

```bash
databricks bundle run environment_bootstrap --target dev
databricks bundle run environment_bootstrap --target dev   # should succeed identically
```

This is the same "test the thing that's supposed to almost never fail" discipline [Section 5,
Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
applied to testing `dq_check`'s gate on purpose -- an idempotent job that's never actually been run
twice is only idempotent in theory, and CI is the wrong place to discover otherwise for the first
time.

## Why a direct resource reference beats a variable, restated

It's worth dwelling on why `${resources.pipelines.bronze_cdc.id}` is strictly better than
`${var.bronze_pipeline_id}` here, not just different. A variable requires a human to know the
correct pipeline ID and keep it current in every target's `variables` block -- three separate
values to maintain, three separate places a typo or a stale ID can quietly break a job task without
any validation catching it until the job actually tries to run. A direct resource reference has no
such failure mode: the bundle framework deploys `bronze_cdc` first, learns its actual ID from the
API response, and substitutes that ID into `orchestration_job.yml` automatically, every time,
regardless of target. The variable version made sense back in Section 5 only because no bundle
existed yet to resolve a direct reference against -- once one does, direct references are the
correct default for any resource depending on another resource in the same bundle.

## Why the bootstrap job is separate from the daily pipeline job

`environment_bootstrap` could theoretically live as a fifth task on `stepright_daily_pipeline`,
gated to run only once -- but bundling a rarely-run, environment-setup concern into a job that
otherwise runs every single night at 3 AM would mean every ordinary daily run at least evaluates
whether bootstrap needs to happen, for no benefit on any of the 364 days a year it doesn't. Keeping
it a fully separate job, with no schedule of its own, means the daily pipeline's DAG stays exactly
the four tasks Section 5 designed, and bootstrap runs exactly when someone (or Lecture 6's CI/CD
pipeline) explicitly decides a new environment needs standing up.

## `resources/dq_dashboard.yml`

```yaml
resources:
  dashboards:
    dq_monitoring:
      display_name: "StepRight Data Quality Monitoring"
      file_path: resources/dq_dashboard.lvdash.json
      warehouse_id: ${var.sql_warehouse_id}
```

The `.lvdash.json` file [Section 7, Lecture 4]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/creating-data-quality-monitoring-dashboard/' | relative_url }})
exported sits alongside this thin wrapper, which is what actually makes the dashboard a bundle
resource -- deployed, versioned, and promoted across targets the same way every pipeline and job
here is, rather than a one-off export living outside the bundle's reach.

## The new resource: environment bootstrap

`uat` and `prod` don't have a catalog, schema, or landing volume yet -- Section 1, Lecture 3's
`CREATE CATALOG`/`CREATE SCHEMA`/`CREATE VOLUME` statements only ever ran once, by hand, against
`dev`. A bootstrap job runs the same statements, parameterized, against whichever target actually
needs them:

```yaml
# resources/environment_bootstrap_job.yml
resources:
  jobs:
    environment_bootstrap:
      name: "StepRight Environment Bootstrap"
      tasks:
        - task_key: bootstrap
          notebook_task:
            notebook_path: ./notebooks/bootstrap_environment
            base_parameters:
              catalog: ${var.catalog}
              schema: ${var.schema}
```

```python
# notebooks/bootstrap_environment
catalog = dbutils.widgets.get("catalog")
schema = dbutils.widgets.get("schema")

spark.sql(f"CREATE CATALOG IF NOT EXISTS {catalog} COMMENT 'StepRight {catalog} catalog'")
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog}.{schema} COMMENT 'StepRight bronze, silver, and gold tables'")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {catalog}.{schema}.landing COMMENT 'Governed landing zone'")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {catalog}.{schema}.staging COMMENT 'Test data staging'")

for source in ["products", "inventory", "clickstream", "fulfillment"]:
    dbutils.fs.put(f"/Volumes/{catalog}/{schema}/landing/{source}/.gitkeep", "", overwrite=True)
```

`IF NOT EXISTS` on every statement is what makes this genuinely safe to run repeatedly -- against
`dev`, where the catalog and schema already exist, it's a harmless no-op; against a fresh `uat` or
`prod`, it's exactly what stands the environment up for the first time. This job runs on demand,
not on Section 5's daily schedule -- it has no `schedule` block at all, triggered manually or as
one explicit step in Lecture 6's CI/CD pipeline the first time a new target gets deployed.

## `databricks.yml`, updated with the new variable

```yaml
variables:
  catalog:
    description: "Unity Catalog catalog for this target"
  schema:
    description: "Schema within the catalog"
    default: step_right
  notification_email:
    description: "Email address for job failure notifications"
    default: data-eng-team@company.com
  sql_warehouse_id:
    description: "SQL warehouse the dashboard queries run against"
```

`sql_warehouse_id` has no default -- unlike `schema`, a SQL warehouse ID is genuinely different per
workspace and per target, with no sensible shared value, so each target's `variables` block must
supply its own rather than inheriting a default that would be wrong somewhere.

## Why the dashboard resource needs a warehouse ID and the pipelines don't

A Lakeflow Declarative Pipeline provisions its own serverless compute per Lecture 1 and Section
2's `serverless: true` setting -- nothing external to reference. A dashboard has no compute of its
own; every visualization runs its underlying SQL against a **SQL warehouse**, an existing piece of
infrastructure the dashboard resource has to be told about explicitly via `warehouse_id`. That
warehouse isn't something this bundle creates -- it's assumed to already exist in each target
workspace, the same way the workspace itself and its host URL are assumed to already exist rather
than being something `databricks.yml` provisions from nothing.

## What didn't need to change

Nothing in `transformations/`, `tests/`, or `dq_helpers.py` changes in this lecture -- every
resource file here wraps existing, already-verified pipeline and job logic in bundle syntax without
touching what any of it actually computes. That's worth noting explicitly: packaging a project for
deployment is infrastructure work, and infrastructure work that silently changes business logic
along the way is exactly the kind of scope creep Section 6, Lecture 2 warned against when
refactoring `gold_daily_revenue` -- extract or repackage, verify identical behavior, and treat any
actual logic change as a separate, deliberate step.

## The final shape of `resources/`

```text
resources/
├── pipeline_bronze.yml
├── pipeline_silver_gold.yml
├── orchestration_job.yml
├── environment_bootstrap_job.yml
├── dq_dashboard.yml
└── dq_dashboard.lvdash.json
```

Six files, `include: [resources/*.yml]` picking up every `.yml` among them automatically -- the
`.lvdash.json` itself isn't matched by that glob and doesn't need to be; it's referenced by
`file_path` from `dq_dashboard.yml`, not included as a bundle resource definition in its own right.

## Common mistakes

- **Forgetting `IF NOT EXISTS` on any bootstrap statement.** Without it, a second run of
  `environment_bootstrap` against an environment that already exists fails outright instead of
  safely no-op-ing -- exactly the idempotency property that makes this job safe to include as a
  routine step in Lecture 6's CI/CD pipeline rather than a manual, one-time-only ritual.
- **Referencing `${resources.pipelines.bronze_cdc.id}` from a resource file that doesn't declare
  `bronze_cdc` and isn't in the same bundle.** Cross-resource references only resolve within one
  `databricks.yml`'s combined set of included files -- this works precisely because `include:
  [resources/*.yml]` pulls every file in this lecture into one bundle, not because the reference
  syntax reaches across bundles.
{: .important }

## What's next

Every resource this project has built across eight sections now lives in `resources/`, bundle-managed,
environment-aware. Lecture 3 actually deploys to `dev` for the first time -- including the one real
wrinkle every one of these pipelines and the job already existing in `dev`, created by hand back in
Sections 2 and 5, creates for a first bundle deploy.

<!-- prevnext:start -->

---

| [&larr; Previous: Package your project using Declarative Automation Bundle]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/package-your-project-using-declarative-automation-bundle/' | relative_url }}) | [Next: Test Your Deployment Bundle in Dev Environment &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/test-your-deployment-bundle-in-dev-environment/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

