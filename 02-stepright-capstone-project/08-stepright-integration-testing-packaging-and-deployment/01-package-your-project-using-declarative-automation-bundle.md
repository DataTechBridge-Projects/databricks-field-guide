---
title: "Package your project using Declarative Automation Bundle"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 1
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/package-your-project-using-declarative-automation-bundle/
read_minutes: 18
---

# Package your project using Declarative Automation Bundle
{: .no_toc }

*Estimated read: 18 min*

Seven sections in, `resources/` holds four things: Section 2's bronze `pipeline.yml`, Section 5's
silver-gold pipeline definition, Section 5's `orchestration_job.yml`, and Section 7's
`dq_dashboard.lvdash.json`. Every one of them was created by hand -- `databricks pipelines create`,
`databricks jobs run-now`, a UI click-through -- against a single `dev` catalog, with no concept of
`uat` or `prod` anywhere in the project. This lecture introduces **Databricks Asset Bundles** (the
official name for what the manifest calls a "Declarative Automation Bundle") -- the mechanism that
turns those four hand-deployed pieces into one versioned, environment-aware deployable unit.

## What a bundle actually is

A **Databricks Asset Bundle** is infrastructure-as-code for Databricks: a `databricks.yml` root
manifest declaring every resource a project owns -- pipelines, jobs, dashboards -- plus one or more
**deployment targets**, each mapping the same resource definitions onto a different workspace,
catalog, and set of overrides. `databricks bundle deploy --target dev` and `databricks bundle
deploy --target uat` run the exact same YAML through two different lenses, rather than two
independently maintained sets of pipeline and job configuration that inevitably drift apart from
each other over time.

This is the direct Databricks-native answer to a problem every legacy ETL team already knows by a
different name: a Talend job promoted from dev to UAT to prod by manually re-pointing connection
strings and re-exporting `.zip` files at each stage, with no guarantee the version that reached
prod matches what was actually tested in UAT. A bundle's `databricks.yml`, checked into
`steprightproject`'s Git history alongside every transformation file this course has built, is the
single source of truth for what "the StepRight pipeline" means in any environment -- diffable,
reviewable, and revertible the same way every other file in this repo already is.

## Why now, and not from Section 2

Building `pipeline.yml` and `orchestration_job.yml` by hand first, then packaging them here, wasn't
an oversight -- it mirrors how a real legacy-to-Databricks migration often actually proceeds: prove
a pipeline works, prove the job orchestrates it correctly, prove the dashboard reads the right
data, and only then invest in the infrastructure-as-code layer that makes deploying all of it
repeatable across environments. Introducing a bundle before any of Sections 2 through 7 existed
would have meant learning Asset Bundle syntax and Lakeflow pipeline design at the same time, with
no working pipeline yet to anchor either one against.

## Anatomy of `databricks.yml`

```yaml
# databricks.yml
bundle:
  name: steprightproject

include:
  - resources/*.yml

variables:
  catalog:
    description: "Unity Catalog catalog for this target"
  schema:
    description: "Schema within the catalog"
    default: step_right
  notification_email:
    description: "Email address for job failure notifications"
    default: data-eng-team@company.com

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://your-workspace.cloud.databricks.com
    variables:
      catalog: dev

  uat:
    mode: production
    workspace:
      host: https://your-workspace.cloud.databricks.com
    variables:
      catalog: uat
    run_as:
      service_principal_name: stepright-uat-sp

  prod:
    mode: production
    workspace:
      host: https://your-workspace.cloud.databricks.com
    variables:
      catalog: prod
    run_as:
      service_principal_name: stepright-prod-sp
```

Four top-level blocks, each doing one job: `bundle` names the project; `include` tells the CLI
where to find resource definitions (every `.yml` file already sitting in `resources/`, picked up
automatically, the same glob-include instinct Section 2's `pipeline.yml` used for its own
transformation files); `variables` declares the knobs every target can override; `targets` is
where `dev`, `uat`, and `prod` each supply their own values for those knobs.

## What `databricks bundle deploy` actually does

Running `deploy` isn't a metaphor for "make the workspace match the YAML" -- it's a concrete
sequence: the CLI uploads every file the bundle references (transformation code, notebooks, the
resource definitions themselves) into a workspace file path under
`/Workspace/Users/<deploying-identity>/.bundle/steprightproject/<target>/files`, then calls the
Databricks REST API to create or update each declared resource (a pipeline, a job, a dashboard) so
it points at that uploaded code. A second `deploy` against the same target is idempotent -- it
diffs the current deployed state against the YAML and only changes what actually differs, the same
way `terraform apply` or `git push` only moves what changed, not a full teardown-and-recreate every
time.

## One resource file per concern, not one giant `databricks.yml`

`include: [resources/*.yml]` is what makes it possible to keep `resources/pipeline_bronze.yml`,
`resources/pipeline_silver_gold.yml`, `resources/orchestration_job.yml`, and
`resources/dq_dashboard.yml` as separate files rather than pasting every resource definition into
one sprawling `databricks.yml`. This mirrors the same one-file-per-concern discipline
`transformations/` already follows -- `gold_reporting.py` and `silver_files.py` stay separate files
for the same reason a pipeline resource and a job resource stay in separate YAML files: each is
independently readable, and a diff touching only the job doesn't show up as noise in a pipeline
file's review.

## `${var.X}` substitution, beyond `catalog`

Bundle variable substitution isn't limited to catalog names -- any string value in any resource
file under `resources/` can reference `${var.<name>}`, `${bundle.target}` (the active target's
name, useful for a resource's display name), or `${workspace.current_user.userName}` (useful in
`mode: development` contexts). A few examples worth previewing before Lecture 2 rewrites every
resource file in full:

```yaml
# A job's notification target, driven by the notification_email variable
email_notifications:
  on_failure: ["${var.notification_email}"]

# A schema reference built from two variables at once
catalog: ${var.catalog}
schema: ${var.schema}

# A resource name that shows which target deployed it, useful where mode: development's
# automatic per-user prefixing doesn't apply (uat and prod both run in mode: production)
name: "stepright_daily_pipeline_${bundle.target}"
```

Every one of these resolves at `deploy` time, per target -- the exact same resource file produces
`catalog: uat` when deployed to `uat` and `catalog: prod` when deployed to `prod`, with zero
hand-editing between the two.

## Targets can point at the same workspace or different ones

Nothing about a bundle target requires a separate Databricks workspace per environment -- StepRight
keeps all three targets pointed at the same `workspace.host` in the example above, relying on
Unity Catalog's `dev`/`uat`/`prod` *catalogs* for isolation rather than separate workspaces
entirely. A larger organization with stricter network or compliance boundaries between
environments would instead give each target its own `workspace.host`, pointing at genuinely
separate workspaces -- the YAML shape is identical either way; only the `host` value under each
target's `workspace` block changes. StepRight's single-workspace, catalog-isolated approach is the
simpler, and for a project this size, entirely sufficient choice.

## Guarding against an accidental `prod` deploy

`databricks bundle deploy` with no `--target` flag deploys to whichever target is marked
`default: true` -- `dev` here, deliberately, so a bare `deploy` command run out of habit can never
accidentally reach `prod`. Reaching `uat` or `prod` always requires spelling out
`--target uat` or `--target prod` explicitly, a small but real safeguard against muscle memory
causing a production deploy nobody intended to run.

## `catalog` and `schema` as bundle variables

`schema` stays `step_right` in every target -- Section 1, Lecture 3 already established that the
schema, not the catalog, is what isolates this project from everything else in the workspace.
`catalog`, by contrast, is exactly what changes: `dev` for the target used throughout Sections 2
through 7, `uat` and `prod` as two catalogs that don't exist yet anywhere in the workspace.
Deploying to a new target for the first time is what actually creates them -- Lecture 2 covers the
bootstrap resource that runs the `CREATE CATALOG`/`CREATE SCHEMA`/`CREATE VOLUME` statements from
Section 1, Lecture 3 against whichever catalog name the active target supplies.

## `mode: development` vs. `mode: production`

| | `mode: development` (dev) | `mode: production` (uat, prod) |
|---|---|---|
| Resource naming | Prefixed with the deploying user's identity (`[dev alice] stepright_daily_pipeline`) | Exact name, no prefix |
| Pipeline development setting | `development: true` by default -- faster iteration, no retries on failure | `development: false` -- full retry and monitoring behavior |
| Concurrent deploys from different developers | Expected and safe -- each developer's prefix keeps their deployed resources separate | Not expected -- one canonical deployment per target |
| Typical `run_as` | The deploying user | A service principal, not a personal identity |

`mode: development`'s automatic per-user prefixing is what lets multiple engineers each run
`databricks bundle deploy --target dev` against the same workspace without colliding -- a second
developer's `dev` deploy creates their own separately-named pipeline and job, not a conflicting
redeploy of the first developer's resources. `uat` and `prod` deliberately opt out of that
behavior: there should be exactly one `stepright_daily_pipeline` in each of those catalogs, owned
by a service principal, not by whichever engineer happened to run the deploy command last.

## Why `run_as` matters for `uat` and `prod`

A job or pipeline deployed under a personal identity breaks the moment that person leaves the team,
changes roles, or simply has their laptop credentials expire -- a **service principal**
(`stepright-uat-sp`, `stepright-prod-sp`) is a non-human identity that owns the deployed resources
instead, the same reasoning behind using a service principal for any production automation rather
than a person's own login. `dev` skips this deliberately -- individual engineer identity is exactly
what `mode: development` wants for its per-user resource prefixing to make sense in the first
place.
{: .important }

## A preview: one resource file, before and after

Lecture 2 rewrites all four resource files in full; one worked example here makes the pattern
concrete. Section 2, Lecture 2 left `pipeline.yml` with catalog and schema hardcoded:

```yaml
# Before -- Section 2, Lecture 2
name: steprightproject-bronze-cdc
catalog: dev
schema: step_right
serverless: true
continuous: false
libraries:
  - glob:
      include: transformations/bronze_*.py
```

As a bundle resource, the same pipeline becomes:

```yaml
# resources/pipeline_bronze.yml
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

Two changes, both small: `catalog: dev` and `schema: step_right` become `${var.catalog}` and
`${var.schema}`, and the whole definition nests one level deeper, under `resources: pipelines:
bronze_cdc:` -- that nesting is what gives this pipeline a **resource key** (`bronze_cdc`), the
name other resources in the same bundle use to reference it directly, which matters the moment
`orchestration_job.yml` needs this pipeline's ID.

## Referencing one bundle resource from another

```yaml
# resources/orchestration_job.yml (excerpt)
tasks:
  - task_key: run_ingestion
    pipeline_task:
      pipeline_id: ${resources.pipelines.bronze_cdc.id}
```

`${resources.pipelines.bronze_cdc.id}` resolves to whatever pipeline ID the bundle actually created
for that resource key, in that target -- the bundle framework tracks this automatically, so the job
resource never needs to know or hardcode an actual pipeline ID at all. This is the more idiomatic
alternative to the `${var.bronze_pipeline_id}` placeholder [Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
used before a bundle existed to resolve it against -- Lecture 2 makes this exact substitution for
real.

## Installing and authenticating the CLI

```bash
databricks --version   # v0.230.0 or newer required for full bundle support
databricks auth login --host https://your-workspace.cloud.databricks.com
```

The Databricks CLI is what interprets `databricks.yml` and turns it into actual API calls against a
workspace -- `databricks bundle deploy`, `validate`, `run`, and `destroy` are the four commands
this section leans on most, covered as Lecture 3 actually deploys for the first time.

## Validating before deploying anything

```bash
databricks bundle validate --target dev
```

`validate` parses `databricks.yml` and every included resource file, checks for schema errors and
unresolved variable references, and reports them without touching the workspace at all -- the
bundle equivalent of a syntax check, worth running after every edit to any file under `resources/`
or to `databricks.yml` itself, well before ever running `deploy`.

## Inspecting and rolling back a deployment

```bash
databricks bundle summary --target dev      # what's currently deployed, and where
databricks bundle open --target dev         # opens the deployed job/pipeline in the browser
databricks bundle destroy --target dev      # tears down every resource this bundle deployed
```

`summary` is the fastest way to answer "what does `dev` actually have deployed right now" without
hunting through the Jobs and Pipelines UI by hand. `destroy` is the deliberate inverse of `deploy`
-- it removes every resource the bundle created in that target, which matters for `dev` specifically:
a developer's per-user-prefixed `dev` resources are meant to be disposable, torn down and
redeployed freely while iterating, in a way `uat` and `prod` resources never should be.

## Secrets never belong in `databricks.yml`

Nothing this lecture's `databricks.yml` example needs is actually a secret -- catalog names, an
email address, a service principal's *name* are all fine to commit in plain text. Credentials
themselves (a service principal's client secret, an API token used by CI) are a different matter,
and belong in a secrets manager or CI provider's own secret store, injected as an environment
variable at deploy time rather than written into any file this Git folder tracks. Lecture 6's
CI/CD pipeline covers exactly where those credentials actually live and how they reach the
`databricks bundle deploy` command without ever touching version control.

## What existed before this lecture, and what changes now

| | Before (Sections 2-7) | After (this section) |
|---|---|---|
| How a pipeline gets created | `databricks pipelines create --json @pipeline.yml`, by hand | `databricks bundle deploy`, one command for every resource |
| Environment awareness | None -- everything hardcoded to `dev.step_right` | `dev`, `uat`, `prod` targets, one shared resource definition |
| Who owns deployed resources | Whoever ran the CLI command | A service principal, in `uat` and `prod` |
| Change history | Whatever the Git folder's commit log shows for the YAML files themselves | The same, plus a reviewable diff of what actually gets deployed on every change |

Nothing about the pipelines, the job's DAG, or the dashboard's queries changes here -- this lecture
packages what already works, it doesn't redesign it.

## Why this matters far beyond one capstone project

A promotion path this clean -- one shared resource definition, three targets, a service principal
owning anything beyond `dev` -- is exactly the piece a legacy-to-Databricks migration needs once
pipelines move past a single engineer's proof of concept. A migration project with dozens of
translated ETL jobs, each needing its own dev-to-prod promotion story, either builds this pattern
once and reuses it everywhere, or reinvents ad hoc promotion scripts per pipeline -- the same
"many small consistent pieces over one clever combined solution" instinct this course has applied
repeatedly, now applied at the level of an entire project's deployment story rather than one
function or one view.

## Common mistakes

- **Hardcoding `dev.step_right` inside a resource YAML file instead of referencing the `catalog`
  variable.** A pipeline definition with `catalog: dev` written literally, rather than `catalog:
  ${var.catalog}`, silently ignores whichever target actually gets deployed -- Lecture 2 covers
  exactly this substitution for every resource file `resources/` holds.
- **Skipping `databricks bundle validate` and going straight to `deploy`.** A YAML syntax error or
  an unresolved variable reference is far cheaper to catch locally, in seconds, than mid-deploy
  against a real workspace.
- **Running `databricks bundle destroy` against `uat` or `prod` out of habit from `dev` iteration.**
  `destroy` removes every resource the bundle owns in that target -- muscle memory built up
  tearing down and redeploying disposable `dev` resources is exactly the habit that makes this
  command dangerous the one time `--target` gets typed without thinking in a production context.

## What's next

Lecture 2 rewrites Section 2, Section 5, and Section 7's resource files to use `${var.catalog}`
and `${var.schema}` throughout, adds the environment bootstrap job this lecture referenced, and
reconciles Section 5's `${var.bronze_pipeline_id}` placeholder with how bundle-managed pipelines
actually reference each other.

<!-- prevnext:start -->

---

| [&larr; Previous: StepRight - Integration Testing, Packaging and Deployment]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/' | relative_url }}) | [Next: Define your Automation Bundle Resources and Setup Job &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/define-your-automation-bundle-resources-and-setup-job/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

