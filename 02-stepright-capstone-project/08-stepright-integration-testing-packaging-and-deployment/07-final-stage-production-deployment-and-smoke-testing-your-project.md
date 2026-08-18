---
title: "Final Stage - Production Deployment and Smoke Testing Your Project"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 7
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/final-stage-production-deployment-and-smoke-testing-your-project/
read_minutes: 10
---

# Final Stage - Production Deployment and Smoke Testing Your Project
{: .no_toc }

*Estimated read: 10 min*

`deploy-prod` is waiting on a reviewer's approval. This closing lecture covers what happens the
moment that approval lands -- a fast **smoke test**, not a rerun of Lecture 5's full integration
suite, followed by the first real production run and what actually needs watching afterward.

## Smoke test, not integration test: a deliberate distinction

Lecture 6 already established why the full integration suite never runs against `prod` --
`test_dq_check_failure_skips_transformation`'s deliberately-broken seeded batch would be actively
harmful against real data. A **smoke test** asks a much narrower question: did the deploy itself
succeed, not does every piece of business logic work correctly (already proven twice, in unit
tests and in `uat`'s integration suite). "Does the system turn on" is the entire scope -- named for
the literal hardware-engineering practice of powering on a newly assembled circuit and checking
for smoke before testing anything more specific.

## The smoke test script

```python
# scripts/smoke_test.py
import os
import sys
from databricks.sdk import WorkspaceClient

def main():
    client = WorkspaceClient(
        host=os.environ["DATABRICKS_HOST"], token=os.environ["DATABRICKS_TOKEN"]
    )
    checks = []

    # 1. Catalog, schema, and volumes exist
    try:
        client.schemas.get(full_name="prod.step_right")
        checks.append(("schema exists", True))
    except Exception as e:
        checks.append(("schema exists", False, str(e)))

    # 2. Both pipelines are deployed and in a healthy state
    for name in ["steprightproject-bronze-cdc", "steprightproject-silver-gold"]:
        pipelines = list(client.pipelines.list_pipelines(filter=f"name LIKE '{name}'"))
        checks.append((f"pipeline '{name}' exists", len(pipelines) == 1))

    # 3. The daily job exists and is scheduled
    jobs = list(client.jobs.list(name="StepRight Daily Pipeline"))
    checks.append(("daily job exists", len(jobs) == 1))
    if jobs:
        checks.append(("daily job has a schedule", jobs[0].settings.schedule is not None))

    # 4. The dashboard resource deployed successfully
    dashboards = list(client.lakeview.list())
    checks.append((
        "dq monitoring dashboard exists",
        any(d.display_name == "StepRight Data Quality Monitoring" for d in dashboards),
    ))

    failures = [c for c in checks if not c[1]]
    for check in checks:
        status = "PASS" if check[1] else "FAIL"
        print(f"[{status}] {check[0]}")

    if failures:
        sys.exit(1)
    print("Smoke test passed -- prod deployment is healthy.")

if __name__ == "__main__":
    main()
```

Four categories, each answering "does this exist and look right" -- never "is this specific number
correct," which is exactly the line separating a smoke test from an integration test. This runs in
seconds, not the many minutes Lecture 5's suite needs, since it never triggers an actual pipeline
run at all.

## Wiring it into `deploy-prod`

```yaml
      - name: Deploy bundle to prod
        env:
          DATABRICKS_HOST: ${{ secrets.PROD_DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.PROD_DATABRICKS_TOKEN }}
        run: |
          databricks bundle validate --target prod
          databricks bundle deploy --target prod
          databricks bundle run environment_bootstrap --target prod
      - name: Smoke test prod deployment
        env:
          DATABRICKS_HOST: ${{ secrets.PROD_DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.PROD_DATABRICKS_TOKEN }}
        run: python scripts/smoke_test.py
```

A failed smoke test here fails the whole `deploy-prod` job -- worth alerting on immediately, since
it means the deploy itself is broken, not that some downstream business rule needs a closer look.

## The first real production run

```bash
databricks bundle run stepright_daily_pipeline --target prod
```

Triggered manually, once, rather than waiting for the 3 AM schedule to fire on its own -- watching
this first `prod` run complete end to end, with real eyes on the Jobs UI, is worth doing once
before trusting the schedule alone. Confirm `report`'s printed summary shows real production
counts, not the small, synthetic batches `uat`'s integration suite seeds.

## What actually needs watching after go-live

Nothing new gets built here -- every piece already exists from earlier sections, now finally
pointed at real production data:

| What | Where | Built in |
|---|---|---|
| Daily pipeline failure | Email to `data-eng-team@company.com` | Section 5 |
| Quarantine rate trending up | Dashboard + alert | Section 7 |
| A single-run threshold breach | `dq_check`'s gate, plus the breach alert | Sections 5 and 7 |
| Deploy-time failures | GitHub Actions run status, this section | Section 8 |

Go-live doesn't add new monitoring surface area -- it's the moment every monitoring surface this
capstone already built starts watching data that actually matters to the business, rather than
`dev`'s development batches or `uat`'s deliberately-seeded test data.

## Why the smoke test checks existence and configuration, not data

Every check in `smoke_test.py` asks "does this object exist and look configured correctly" --
never "is today's row count reasonable," which is deliberately Section 5's `dq_check`'s job, not
this script's. A smoke test that queried table row counts would duplicate a check that already
runs, automatically, every single day the pipeline executes on its own schedule -- there's no need
to re-verify at deploy time something the daily job already re-verifies at 3 AM regardless. Keeping
the smoke test scoped to deployment health, and nothing broader, is what keeps it fast enough to
run on every single `prod` deploy without becoming a second integration suite in disguise.

## Rolling back a bad `prod` deploy

```bash
git log --oneline -- databricks.yml resources/
git checkout <previous-good-commit>
databricks bundle deploy --target prod
```

A bundle deploy is, at its core, a deployment of whatever code and configuration a specific Git
commit holds -- rolling back means checking out an earlier known-good commit and redeploying it,
the same `bundle deploy` command as any forward deploy, just pointed at older code. This is exactly
why every resource file and every transformation change in this project has lived in version
control from Section 1, Lecture 2 onward: a rollback with no reliable prior version to redeploy
isn't a rollback at all, just a scramble to reconstruct what used to work.
{: .important }

## What this capstone actually proved

Eight sections ago, this project started with an empty Git folder and a plan. What exists now:
seven bronze and silver tables built on Auto Loader and `AUTO CDC`, five gold materialized views
serving five real consumers, a scheduled job with a real data quality gate, a unit-tested
transformation layer, a monitoring dashboard built on the pipeline's own event log, and a full
`dev`-to-`uat`-to-`prod` promotion path with automated deployment and testing at every stage.
Nothing here was theoretical -- every table, every test, every YAML file exists because an earlier
lecture built it for a stated reason, and every later section depended on that earlier work being
genuinely correct, not merely plausible-looking.

## Where this leads

StepRight was built from a clean slate -- no legacy system to migrate, no existing PL/SQL to
translate, no cutover to plan around live production traffic. Part 3 removes that simplification:
a real Oracle-or-Teradata-scale legacy warehouse, decades of stored procedures, and a migration
that has to happen without ever taking the business offline. Every pattern this capstone built --
medallion layering, quality gating, orchestration, testing, and now deployment -- is the target
architecture Part 3 migrates *toward*; what's missing is everything Part 3 adds: how to profile a
system nobody fully understands anymore, how to decide what to rehost versus re-architect, and how
to prove the new system produces the same numbers as the old one before anyone's allowed to turn
the legacy system off.

<!-- prevnext:start -->

---

| [&larr; Previous: Develop and Trigger Your CI/CD Pipeline for Deployment and Integration Testing]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/develop-and-trigger-your-ci-cd-pipeline-for-deployment-and-integration-testing/' | relative_url }}) | [Next: Legacy Migration to Databricks &rarr;]({{ '/03-legacy-migration-to-databricks/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

