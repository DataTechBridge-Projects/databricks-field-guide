---
title: "Planning and Designing Integration Testing Approach"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/planning-and-designing-integration-testing-approach/
read_minutes: 6
---

# Planning and Designing Integration Testing Approach
{: .no_toc }

*Estimated read: 6 min*

`dev` is bundle-managed and verified. Before deploying to `uat` for real, this lecture designs what
**integration testing** actually checks for StepRight -- the layer [Section 6, Lecture 1's testing
pyramid]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/design-test-strategy-for-the-project/' | relative_url }})
named but deliberately left for this section to build.

## What integration tests check that unit tests structurally can't

Section 6's suite proves `compute_daily_revenue` and `dedupe_latest` are correct, in isolation,
against known inputs -- entirely independent of whether the *deployed* pipeline is wired together
correctly. A few real failure modes live in exactly that gap:

- **A glob pattern that doesn't match what it's supposed to.** `libraries: glob: include:
  transformations/silver_*.py` (Lecture 2) matching zero files, or the wrong files, because of a
  typo -- something no unit test, which imports Python modules directly, would ever exercise.
- **A cross-resource reference that resolves to the wrong ID.** `${resources.pipelines.bronze_cdc.id}`
  pointing at a pipeline that doesn't actually exist in a given target, a mistake unit tests have
  no way to catch since they never touch a deployed bundle at all.
- **A Unity Catalog permission that's missing in `uat` but happened to exist in `dev`.** The
  service principal `stepright-uat-sp` runs under (Lecture 1) needs explicit grants on `uat.step_right`
  that were never required in `dev`, where a personal identity with broader default access ran
  everything.
- **The DQ gate actually gating, end to end, in a real job run.** [Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
  tested this once, by hand, in `dev` -- integration testing is what makes that check repeatable on
  every deploy, not a one-time manual verification nobody re-runs.

## What stays explicitly out of scope

Business logic correctness -- discount math, dedup behavior, threshold decisions -- is Section 6's
job, already covered, and integration tests don't re-verify it. Duplicating that coverage here
would mean two suites need updating every time transformation logic changes, with no clear owner
for either. Integration tests exist for the question unit tests can't answer: does the *deployed*
system, wired together the way Lecture 2 and 3 wired it, actually work.
{: .important }

## Test data: a dedicated UAT batch, not `dev`'s accumulated history

`uat.step_right` starts empty after Lecture 3's bootstrap job runs against it -- integration tests
need their own deterministic batch, reusing [Section 1, Lecture 5's Faker generator]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/seed-the-project-data-generator-and-loader-notebook/' | relative_url }})
with a small, fixed seed rather than `dev`'s months of organically accumulated batches. A fixed
seed means every CI run generates the exact same customers, orders, and known injection rates
(the same ~1-2% `unknown_customer_id` and orphaned `order_items` Section 2 originally tuned for),
which is what makes asserting a specific expected quarantine count in a test meaningful rather than
asserting against a number that changes on every run.

## Databricks Connect: the mechanism, briefly revisited

[Section 6, Lecture 4]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/code-and-run-all-unit-test-case/' | relative_url }})'s
comparison table already drew this line: Databricks Connect is what lets a test running from a
laptop or a CI runner execute against a real cluster and query real Unity Catalog tables in
`uat.step_right`, the opposite trade from local PySpark's in-memory, credential-free speed.
Integration tests accept that slower, credential-requiring cost deliberately, because the question
they're answering -- does the deployed system work -- has no answer local PySpark could ever give.

## Why `uat`, and not `dev`, is where this suite runs

`dev`'s data has been accumulating, ad hoc, since Section 1 -- multiple developers' bind
experiments, manual test runs, and whatever batches happened to get loaded across seven sections of
writing this capstone. `uat` starts clean, bootstrapped fresh by Lecture 3's job, seeded only by
whatever this integration suite deliberately puts there -- exactly the controlled, reproducible
starting point a test suite needs, and exactly what `dev`'s organic history can no longer offer
after this much accumulated use. This is also the same environment Lecture 6's CI/CD pipeline
promotes code *into* before it's ever allowed anywhere near `prod`, which makes `uat` the natural,
and only sensible, place for this suite to live.

## The pyramid, one layer fuller

[Section 6, Lecture 1's pyramid diagram]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/design-test-strategy-for-the-project/' | relative_url }})
drew three layers with the middle one still empty. This lecture fills it in for real: dozens of
fast unit tests at the base, a smaller, slower integration suite here in the middle, and Section
5's single job-level DQ gate at the top, watching real production data no test suite could ever
substitute for.

## What Lecture 5 actually builds

A `tests_integration/` suite, run via `pytest` but against a live `uat` deployment rather than
local fixtures: trigger `stepright_daily_pipeline` for real, wait for completion, then assert
against the actual tables and job run metadata that real run produced -- table row counts landing
in the expected range, the DQ gate skipping `run_transformation` when seeded with a deliberately
bad batch, and the dashboard's views returning non-empty results.

## What's next

Lecture 5 writes that suite in full.

<!-- prevnext:start -->

---

| [&larr; Previous: Test Your Deployment Bundle in Dev Environment]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/test-your-deployment-bundle-in-dev-environment/' | relative_url }}) | [Next: Developing your Integration Test for the UAT &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/developing-your-integration-test-for-the-uat/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

