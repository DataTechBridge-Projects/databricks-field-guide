---
title: "Automating Jobs - REST API and Databricks CLI"
parent: "Lakeflow Jobs - The Orchestration Pillar"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/automating-jobs-rest-api-and-databricks-cli/
read_minutes: 14
---

# Automating Jobs - REST API and Databricks CLI
{: .no_toc }

*Estimated read: 14 min*

Everything built through the UI so far in this section can also be created, updated, and triggered
programmatically -- essential once job management needs to live in version control and deploy
through CI/CD, rather than being clicked together by hand in each environment.

## The Databricks CLI

```bash
# List jobs
databricks jobs list

# Get a job's full definition
databricks jobs get --job-id 12345

# Create a job from a JSON definition
databricks jobs create --json @job_config.json

# Trigger a run manually
databricks jobs run-now --job-id 12345 --json '{"job_parameters": {"run_date": "2026-08-14"}}'

# Check a run's status
databricks jobs get-run --run-id 67890
```

The CLI wraps the same REST API calls below in a more scriptable, shell-friendly interface --
generally the right default for one-off operations and CI/CD scripts, over hand-crafting raw HTTP
requests.

## The REST API directly

```python
import requests

response = requests.post(
    f"{DATABRICKS_HOST}/api/2.2/jobs/run-now",
    headers={"Authorization": f"Bearer {DATABRICKS_TOKEN}"},
    json={
        "job_id": 12345,
        "job_parameters": {"run_date": "2026-08-14"}
    }
)
run_id = response.json()["run_id"]
```

The REST API is what the CLI, the workspace UI, and any custom automation all ultimately call --
reaching for it directly makes sense when you need programmatic control from outside a shell
script (a Python-based backfill runner, as in the previous lecture, or a custom monitoring tool).

## Deploying jobs via Databricks Asset Bundles

```yaml
# databricks.yml
bundle:
  name: steprightproject

resources:
  jobs:
    daily_medallion_pipeline:
      name: "Daily Medallion Pipeline"
      schedule:
        quartz_cron_expression: "0 0 2 * * ?"
      tasks:
        - task_key: run_pipeline
          pipeline_task:
            pipeline_id: ${resources.pipelines.medallion_pipeline.id}

targets:
  dev:
    workspace:
      host: https://dev-workspace.cloud.databricks.com
  prod:
    workspace:
      host: https://prod-workspace.cloud.databricks.com
```

```bash
databricks bundle deploy --target dev
databricks bundle deploy --target prod
```

**Key term:** a **Databricks Asset Bundle** is a YAML-defined package of jobs, pipelines, and other
workspace resources, deployable identically across environments via `--target` -- the direct
mechanism that makes "deploy the same job definition to dev, then uat, then prod" a real,
repeatable command rather than manually recreating a job by hand in each workspace's UI. This is
exactly what
[Part 2's StepRight deployment section]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/' | relative_url }})
builds a full CI/CD pipeline around.
{: .important }

## Why this matters for a promotion pipeline

Combined with Git folders (Section 4) and Unity Catalog's environment-scoped catalogs (Section 6),
asset bundles complete the promotion story: code lives in Git, deploys identically to each
environment's own catalog via bundle targets, and runs under each environment's own service
principal (Section 6's identity lecture) -- a coherent dev -> uat -> prod pipeline with no manual,
per-environment configuration drift.

## When to reach for API/CLI automation vs. the UI

| Use the UI when | Use CLI/API/bundles when |
|---|---|
| Initial exploration, one-off jobs, learning | Anything meant to be version-controlled |
| A quick manual trigger or status check | Deploying the same job across multiple environments |
| Debugging a specific run interactively | CI/CD pipelines, scheduled backfills, programmatic monitoring |

For a genuinely production-bound job, treat the UI as a place to *view and debug*, not the source
of truth for *definition* -- the YAML/bundle definition in Git is the source of truth, deployed
consistently rather than hand-edited per environment.

<!-- prevnext:start -->

---

| [&larr; Previous: Multi-task DAGs - Control Flow, Retries, and Backfill]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/multi-task-dags-control-flow-retries-and-backfill/' | relative_url }}) | [Next: Jobs Design Decisions and Production Patterns &rarr;]({{ '/01-databricks-fundamentals/10-lakeflow-jobs-the-orchestration-pillar/jobs-design-decisions-and-production-patterns/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
