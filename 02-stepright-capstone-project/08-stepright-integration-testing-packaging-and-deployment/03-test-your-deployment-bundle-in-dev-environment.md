---
title: "Test Your Deployment Bundle in Dev Environment"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/test-your-deployment-bundle-in-dev-environment/
read_minutes: 12
---

# Test Your Deployment Bundle in Dev Environment
{: .no_toc }

*Estimated read: 12 min*

Every resource file exists. Deploying them naively into `dev` right now would be a mistake --
`dev` already has a bronze pipeline, a silver-gold pipeline, and a daily job, all created by hand
back in Sections 2 and 5. This lecture deploys for real, starting with the one step that keeps
that first deploy from quietly duplicating everything already running.

## The duplication problem, stated plainly

`databricks bundle deploy --target dev`, run with no further preparation, doesn't know
`steprightproject-bronze-cdc` already exists as a hand-created pipeline in `dev` -- as far as the
bundle framework is concerned, `bronze_cdc` is a resource key it has never deployed before, so it
creates a *new* pipeline with a *new* pipeline ID. The result: two pipelines named
`steprightproject-bronze-cdc` in the same catalog, one hand-created back in Section 2 with real
run history, one bundle-managed and empty, and Section 5's job still pointing at the original by
its old, now-orphaned ID.

## `databricks bundle deployment bind`

**Bind** attaches a bundle resource key to an *existing* Databricks object by ID, so the next
`deploy` updates that object in place instead of creating a duplicate:

```bash
# Find the existing resources' real IDs
databricks pipelines list-pipelines | grep steprightproject-bronze-cdc
databricks pipelines list-pipelines | grep steprightproject-silver-gold
databricks jobs list | grep "StepRight Daily Pipeline"

# Bind each bundle resource key to its existing ID
databricks bundle deployment bind bronze_cdc <bronze-pipeline-id> --target dev
databricks bundle deployment bind silver_gold <silver-gold-pipeline-id> --target dev
databricks bundle deployment bind stepright_daily_pipeline <job-id> --target dev
```

Each `bind` is a one-time reconciliation step, run once per resource, per target -- after binding,
the bundle's local state file remembers that `bronze_cdc` *is* that specific pipeline ID, and every
future `deploy --target dev` updates it rather than creating anything new.
{: .important }

## What binding actually finds

```text
$ databricks pipelines list-pipelines | grep steprightproject-bronze-cdc
a1b2c3d4-5e6f-7890-abcd-ef1234567890  steprightproject-bronze-cdc  RUNNING

$ databricks jobs list | grep "StepRight Daily Pipeline"
987654321  StepRight Daily Pipeline
```

Real IDs, copied directly from these two commands' output, are what actually go into the `bind`
commands above -- not a placeholder, not a guess. This is the one step in this whole lecture that
can't be automated away or scripted blindly: a human needs to look at the list and confirm
`a1b2c3d4-...` really is *the* bronze pipeline from Section 2, not a stale duplicate from an
earlier experiment.

## Where the binding actually lives

`bind` writes its result into the bundle's local deployment state, tracked per target under
`.databricks/bundle/dev/` in the project's working directory -- a `terraform.tfstate`-style record
mapping each resource key to the real object ID it's bound to. This state file is what makes every
subsequent `deploy` idempotent and duplication-free: `deploy` consults it before deciding whether a
given resource key needs a `CREATE` or an `UPDATE` call against the Databricks API. It's local
machine or CI-runner state, not something committed to Git -- Lecture 6 covers how CI/CD keeps this
state consistent across runs without relying on one developer's laptop holding the only copy.

## Undoing an accidental unbound deploy

Running `deploy` before binding -- exactly the mistake this lecture opened with -- is recoverable,
not catastrophic:

```bash
databricks bundle destroy --target dev   # removes the bundle's newly-created duplicates only
databricks bundle deployment bind bronze_cdc <original-pipeline-id> --target dev
databricks bundle deployment bind silver_gold <original-pipeline-id> --target dev
databricks bundle deployment bind stepright_daily_pipeline <original-job-id> --target dev
databricks bundle deploy --target dev
```

`destroy` only removes objects the bundle itself created and is currently tracking in its state
file -- the original, hand-created pipelines and job from Sections 2 and 5 are untouched by it,
since the bundle never bound to them in this scenario. That's what makes this recoverable at all:
clean up the accidental duplicates, then follow the correct bind-first sequence from a clean state.

## Validating, then deploying

```bash
databricks bundle validate --target dev
databricks bundle deploy --target dev
```

`validate` catches YAML and variable-resolution errors before touching the workspace, exactly as
Lecture 1 described. `deploy`, now that every pre-existing resource is bound, updates the three
hand-created objects in place -- same pipeline IDs, same job ID, now pointing at bundle-managed
code and configuration instead of whatever was deployed by hand.

## Confirming nothing duplicated

```bash
databricks bundle summary --target dev
databricks pipelines list-pipelines | grep steprightproject
databricks jobs list | grep "StepRight Daily Pipeline"
```

Each of the last two commands should return exactly one match -- one bronze pipeline, one
silver-gold pipeline, one job, matching the IDs bound above. Two matches for any of them means a
bind step was missed or pointed at the wrong ID, and it's worth catching here, in `dev`, rather
than in Lecture 6's CI/CD pipeline running unattended.

## Running the bootstrap job -- and confirming it's a safe no-op in `dev`

```bash
databricks bundle run environment_bootstrap --target dev
```

`dev`'s catalog, schema, and volumes already exist from Section 1, Lecture 3 -- this run should
complete quickly with every `CREATE ... IF NOT EXISTS` statement finding its target already
present, exactly the idempotent behavior Lecture 2 designed and tested by hand. Confirming that
here, against real `dev` infrastructure, is what makes trusting this same job against a genuinely
empty `uat` catalog later a reasonable bet rather than a hope.

## Running the daily pipeline job through the bundle

```bash
databricks bundle run stepright_daily_pipeline --target dev
```

Watch the same four-task DAG [Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
built originally: `run_ingestion`, `dq_check`, `run_transformation`, `report`, in the same order,
gated the same way. Nothing about *how* the job runs changes because it's now bundle-deployed --
`bundle run` is simply a wrapper around the same `run-now` API call `databricks jobs run-now`
already made, pointed at whichever job ID this target's binding resolved to.

## Confirming the run's output looks exactly like it always did

```bash
databricks jobs get-run <run-id> | grep -A5 '"task_key": "report"'
```

The `report` task's printed summary (Section 5, Lecture 3) should read identically to any prior
run -- same bronze/silver/gold counts for the same day, same format. A bundle-deployed job that
subtly produces different output than the hand-deployed version it replaced would mean something
changed in how the code got packaged or uploaded, not in the business logic itself, and this is the
concrete check that rules that out before trusting the bundle for anything further.

## Why `uat` and `prod` skip the binding step entirely

Binding only matters where a resource already exists outside the bundle's management -- `uat` and
`prod` have never had a pipeline, job, or dashboard created in them at all, hand-built or
otherwise. The first `databricks bundle deploy --target uat` creates everything fresh, with no
duplication risk, because there's nothing pre-existing to collide with. Lecture 6 and 7 cover that
first real `uat` and `prod` deploy; this lecture's binding work was specifically, and only,
necessary because `dev` carried seven sections' worth of hand-built history into this one.

## Common mistakes

- **Binding the wrong pipeline ID.** If `dev` happens to have more than one pipeline matching a
  similar name (a leftover experiment, say), binding to the wrong one silently redirects the
  bundle's management onto an object nobody intended -- always verify the ID against the pipeline's
  actual run history in the UI before binding, not just its name.
- **Deploying to `dev` before binding, "just to see what happens."** This is exactly how the
  duplication problem this lecture opened with actually occurs -- bind first, deploy second, every
  time a resource might already exist outside the bundle's management.

## What's next

`dev` is fully bundle-managed, verified end to end. Lecture 4 designs what integration testing
means for this project -- a different kind of test than Section 6's unit suite, run against this
same deployed `dev` environment before anything gets promoted toward `uat`.

<!-- prevnext:start -->

---

| [&larr; Previous: Define your Automation Bundle Resources and Setup Job]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/define-your-automation-bundle-resources-and-setup-job/' | relative_url }}) | [Next: Planning and Designing Integration Testing Approach &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/planning-and-designing-integration-testing-approach/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

