---
title: "Developing your Integration Test for the UAT"
parent: "StepRight - Integration Testing, Packaging and Deployment"
grand_parent: "StepRight Capstone Project"
nav_order: 5
permalink: /02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/developing-your-integration-test-for-the-uat/
read_minutes: 20
---

# Developing your Integration Test for the UAT
{: .no_toc }

*Estimated read: 20 min*

Lecture 4 designed what to check; this lecture writes it -- a `tests_integration/` suite that
deploys nothing itself, but triggers the real, already-deployed `uat` job, waits for it, and asserts
against what actually landed in `uat.step_right`.

## `tests_integration/conftest.py`: a Databricks Connect fixture, not a local one

```python
# tests_integration/conftest.py
import os
import pytest
from databricks.connect import DatabricksSession
from databricks.sdk import WorkspaceClient

@pytest.fixture(scope="session")
def spark():
    return DatabricksSession.builder.remote(
        host=os.environ["DATABRICKS_HOST"],
        token=os.environ["DATABRICKS_TOKEN"],
        cluster_id=os.environ["DATABRICKS_CLUSTER_ID"],
    ).getOrCreate()

@pytest.fixture(scope="session")
def workspace_client():
    return WorkspaceClient(host=os.environ["DATABRICKS_HOST"], token=os.environ["DATABRICKS_TOKEN"])

@pytest.fixture(scope="session")
def catalog():
    return "uat"
```

Every credential comes from an environment variable, never a hardcoded value -- exactly the
"secrets never belong in `databricks.yml`" rule Lecture 1 established, applied here to the test
suite's own configuration. `DatabricksSession.builder.remote(...)` is Databricks Connect's entry
point: a `spark` object that looks like any other SparkSession to calling code, but every query it
runs actually executes against the real `uat` cluster, not in-process the way Section 6's local
fixture did.

## One new resource this suite depends on: the generator as a job, not a manual notebook

[Section 1, Lecture 5]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/seed-the-project-data-generator-and-loader-notebook/' | relative_url }})'s
Faker generator and loader notebook were always run by hand, interactively, into `dev` -- fine for
a human building this project section by section, but unusable from an automated test suite with
no one present to click "Run all." A small addition to `resources/` wraps the same generator script
as a bundle-managed job, `StepRight Data Generator`, parameterized by `seed` and injection-rate
overrides, so this suite (and Lecture 6's CI/CD pipeline) can trigger it the same programmatic way
it triggers the daily pipeline job itself.

## Seeding `uat` with a deterministic batch

```python
# tests_integration/conftest.py (continued)
@pytest.fixture(scope="session", autouse=True)
def seed_uat_data(workspace_client):
    """Runs once per test session, before any test -- generates a fixed-seed batch into uat's landing volume."""
    workspace_client.jobs.run_now_and_wait(
        job_id=get_job_id(workspace_client, "StepRight Data Generator"),
        job_parameters={"seed": "42", "batch_size": "small"},
    )
```

A fixed `seed` value is what makes this reproducible -- the same customers, orders, and ~1-2%
deliberately injected `unknown_customer_id` and orphaned `order_items` rows, every single run,
which is what lets later assertions check specific expected counts rather than a range wide enough
to be meaningless.

## Test: the daily pipeline job runs successfully end to end

```python
# tests_integration/test_pipeline_run.py
import time

def wait_for_run(workspace_client, run_id, timeout_seconds=900):
    deadline = time.time() + timeout_seconds
    while time.time() < deadline:
        run = workspace_client.jobs.get_run(run_id)
        if run.state.life_cycle_state in ("TERMINATED", "SKIPPED", "INTERNAL_ERROR"):
            return run
        time.sleep(15)
    raise TimeoutError(f"Run {run_id} did not complete within {timeout_seconds}s")

def test_daily_pipeline_completes_successfully(workspace_client):
    job_id = get_job_id(workspace_client, "StepRight Daily Pipeline")
    run = workspace_client.jobs.run_now(job_id=job_id, job_parameters={"run_date": "2026-08-17"})
    result = wait_for_run(workspace_client, run.run_id)
    assert result.state.result_state.value == "SUCCESS"
```

`wait_for_run`'s polling loop, not a blocking call, is the mechanism here -- a real pipeline and job
run takes minutes, not milliseconds, and there's no Databricks API call that blocks until
completion the way a local function call would. Fifteen-second polling intervals balance
responsiveness against not hammering the Jobs API with requests every second for a run that's going
to take several minutes regardless.

## Test: table counts land in the expected range

```python
# tests_integration/test_table_counts.py
def test_bronze_customers_landed(spark, catalog):
    count = spark.sql(f"SELECT count(*) FROM {catalog}.step_right.bronze_customers_valid").collect()[0][0]
    assert count > 0

def test_silver_orders_current_version_matches_valid_bronze_orders(spark, catalog):
    silver_count = spark.sql(
        f"SELECT count(*) FROM {catalog}.step_right.silver_orders WHERE __END_AT IS NULL"
    ).collect()[0][0]
    bronze_count = spark.sql(f"SELECT count(*) FROM {catalog}.step_right.bronze_orders_valid").collect()[0][0]
    assert silver_count == bronze_count

def test_quarantine_rate_matches_known_injection_rate(spark, catalog):
    valid = spark.sql(f"SELECT count(*) FROM {catalog}.step_right.bronze_orders_valid").collect()[0][0]
    quarantined = spark.sql(f"SELECT count(*) FROM {catalog}.step_right.bronze_orders_quarantine").collect()[0][0]
    rate = quarantined / (valid + quarantined)
    assert 0.005 < rate < 0.03  # seed=42's known ~1-2% injection rate, with slack for batch-size variance

def test_gold_customer_360_row_count_matches_silver_customers(spark, catalog):
    gold_count = spark.sql(f"SELECT count(*) FROM {catalog}.step_right.gold_customer_360").collect()[0][0]
    silver_count = spark.sql(
        f"SELECT count(*) FROM {catalog}.step_right.silver_customers WHERE __END_AT IS NULL"
    ).collect()[0][0]
    assert gold_count == silver_count
```

`test_silver_orders_current_version_matches_valid_bronze_orders` is worth pausing on -- it's a
cross-layer consistency check no unit test could ever perform, since Section 6's unit tests feed
`compute_daily_revenue` and `dedupe_latest` synthetic DataFrames directly, never a real bronze
table `silver_orders`'s `AUTO CDC` merge actually consumed.

## Test: the DQ gate actually gates, against a real deliberately-bad run

```python
# tests_integration/test_dq_gate.py
def test_dq_check_failure_skips_transformation(workspace_client):
    # Seed a batch with an inflated clickstream injection rate, deliberately crossing 5%
    workspace_client.jobs.run_now_and_wait(
        job_id=get_job_id(workspace_client, "StepRight Data Generator"),
        job_parameters={"seed": "999", "clickstream_injection_rate": "0.40"},
    )
    job_id = get_job_id(workspace_client, "StepRight Daily Pipeline")
    run = workspace_client.jobs.run_now(job_id=job_id, job_parameters={"run_date": "2026-08-18"})
    result = wait_for_run(workspace_client, run.run_id)

    tasks_by_key = {t.task_key: t for t in result.tasks}
    assert tasks_by_key["dq_check"].state.result_state.value == "FAILED"
    assert tasks_by_key["run_transformation"].state.life_cycle_state.value == "SKIPPED"
```

This is the automated, repeatable version of the manual test [Section 5, Lecture 4]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/creating-the-orchestration-job/' | relative_url }})
ran once by hand -- pushing `clickstream`'s quarantine rate over threshold on purpose and confirming
`run_transformation` shows `SKIPPED`, not `FAILED`, distinguishing "the gate worked as designed"
from "something broke." Running this on every deploy, rather than once during development, is
exactly what catches a future change that accidentally weakens or removes the `run_if:
ALL_SUCCESS` gate before it ever reaches `prod`.

## Test: the dashboard's views return real data

```python
# tests_integration/test_dashboard_views.py
def test_dq_quarantine_rate_view_returns_recent_data(spark, catalog):
    rows = spark.sql(
        f"SELECT * FROM {catalog}.step_right.dq_quarantine_rate WHERE event_date = current_date()"
    ).collect()
    assert len(rows) > 0

def test_dq_referential_orphans_view_has_no_unexpected_check_names(spark, catalog):
    rows = spark.sql(f"SELECT DISTINCT check_name FROM {catalog}.step_right.dq_referential_orphans").collect()
    check_names = {row["check_name"] for row in rows}
    assert check_names.issubset({"unknown_customer_id", "unknown_order_id", "unknown_product_id"})
```

These two matter because [Section 7, Lecture 3]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/prepare-dq-queries-for-expectations-quarantine-and-orphans/' | relative_url }})'s
views were only ever verified by hand, once, against `dev` -- confirming they still resolve
correctly against a freshly bootstrapped `uat` catalog, with different underlying data, is a real
check that a view definition genuinely generalizes rather than happening to work only against the
specific `dev` state it was written and tested with.

## Test isolation: why every seeded batch uses a distinct `run_date`

Two tests in this suite each trigger a real pipeline run against the same `uat.step_right` schema
-- `test_daily_pipeline_completes_successfully` with `run_date=2026-08-17`, and
`test_dq_check_failure_skips_transformation` with `run_date=2026-08-18`. Distinct dates aren't
incidental; they're what keeps the two tests from interfering with each other. Every table this
project has built scopes its quarantine and referential-integrity counts to `date(_ingested_at)`
-- reusing the same `run_date` across two tests would mean the second test's deliberately-inflated
quarantine batch lands on top of the first test's clean batch for the exact same day, corrupting
both tests' assertions at once. Each test owning its own date is the integration-suite equivalent
of Section 6's small, isolated DataFrame fixtures -- different mechanism, same underlying
principle: don't let one test's data bleed into another's.

## Resetting `uat` between full suite runs

A CI run that triggers this suite nightly accumulates one more day's worth of seeded batches every
time, eventually drifting `uat.step_right` away from the clean, bootstrapped state Lecture 3
verified. A lightweight reset, run before the suite rather than after it, keeps every CI run
starting from a known baseline:

```python
# tests_integration/conftest.py (continued)
@pytest.fixture(scope="session", autouse=True)
def reset_uat_tables(spark, catalog):
    tables = [
        "bronze_customers_valid", "bronze_customers_quarantine",
        "bronze_orders_valid", "bronze_orders_quarantine",
        "bronze_order_items_valid", "bronze_order_items_quarantine",
        "bronze_clickstream_valid", "bronze_clickstream_quarantine",
    ]
    for table in tables:
        spark.sql(f"TRUNCATE TABLE {catalog}.step_right.{table}")
```

`TRUNCATE`, not `DROP`, is the deliberate choice -- it clears every row while leaving the table's
schema, permissions, and Lakeflow Declarative Pipelines ownership completely intact, so the
pipeline that owns `bronze_customers_valid` doesn't need to be redeployed or reconciled after every
test run the way dropping and recreating the table would require.

## Handling infrastructure flakiness without hiding real failures

A real cluster occasionally takes longer than expected to spin up; a real API call occasionally
times out for reasons that have nothing to do with StepRight's own code. `pytest-rerunfailures`
gives this suite one narrow, deliberate retry for exactly that class of problem, without silently
masking a genuine failure:

```bash
pytest tests_integration/ -v --timeout=1800 --reruns 1 --reruns-delay 30
```

One rerun, not five -- a test that fails twice in a row against the same seeded data is far more
likely a real bug than transient infrastructure noise, and a suite configured to retry aggressively
risks turning a genuine, reproducible failure into something that eventually passes by luck and
gets reported as green.

## What a real failure looks like

```text
tests_integration/test_dq_gate.py::test_dq_check_failure_skips_transformation FAILED

    def test_dq_check_failure_skips_transformation(workspace_client):
        ...
        tasks_by_key = {t.task_key: t for t in result.tasks}
>       assert tasks_by_key["dq_check"].state.result_state.value == "FAILED"
E       AssertionError: assert 'SUCCESS' == 'FAILED'

tests_integration/test_dq_gate.py:15: AssertionError
================== 1 failed, 11 passed in 743.28s ==================
```

`dq_check` reporting `SUCCESS` against a batch deliberately seeded at a 40% clickstream injection
rate means the gate itself is broken -- either the threshold check regressed, or the seeded batch
didn't actually land the way the test assumed. This is precisely the class of deployment-and-wiring
bug [Section 6, Lecture 1's pyramid]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/design-test-strategy-for-the-project/' | relative_url }})
promised only this layer could catch -- `dq_check`'s own unit tests (Section 6, Lecture 4) already
prove `evaluate_thresholds`'s pure logic is correct; this failure means something about how that
logic is *wired into the deployed job* broke instead, a distinction the failure message alone
doesn't state but the test's placement in this suite, rather than in Section 6's, already implies.

## A shared `get_job_id` helper

```python
# tests_integration/conftest.py (continued)
def get_job_id(workspace_client, job_name):
    for job in workspace_client.jobs.list(name=job_name):
        return job.job_id
    raise ValueError(f"No job found named '{job_name}'")
```

Looking jobs up by name rather than hardcoding an ID keeps this suite portable across whichever
target it happens to run against -- the same job name resolves to a different actual ID in `uat`
versus a future `staging` target, without any test needing to know or care which.

## Test: the `report` task's output actually reflects the run

```python
# tests_integration/test_pipeline_run.py (continued)
def test_report_task_output_mentions_run_date(workspace_client):
    job_id = get_job_id(workspace_client, "StepRight Daily Pipeline")
    run = workspace_client.jobs.run_now(job_id=job_id, job_parameters={"run_date": "2026-08-19"})
    result = wait_for_run(workspace_client, run.run_id)

    report_task = next(t for t in result.tasks if t.task_key == "report")
    output = workspace_client.jobs.get_run_output(report_task.run_id)
    assert "2026-08-19" in output.logs
    assert "GOLD" in output.logs
```

[Section 5, Lecture 3]({{ '/02-stepright-capstone-project/05-stepright-orchestration-and-job-scheduling/job-data-quality-and-health-reporting-in-the-job-logs/' | relative_url }})
built `report` to print into the job's own logs specifically so a human checking on a run doesn't
need a separate tool -- this test confirms that promise holds against a real deployed run, not just
that the task returns a zero exit code. A `report` task that technically succeeds but silently
prints nothing, or prints the wrong date, would pass every other test in this suite while still
failing the one thing `report` exists to do.

## Why this suite tests end to end, not task by task

Every test above triggers the *whole* job and inspects its result, rather than invoking
`run_ingestion`, `dq_check`, `run_transformation`, and `report` as four separately-tested steps.
That's deliberate: the property genuinely worth verifying here is that the deployed *DAG* --
dependencies, `run_if` gating, parameter propagation through `run_date` -- behaves correctly as one
system, which is exactly what four isolated task tests would fail to exercise even if each
individually passed. Testing each task in isolation would edge back toward unit-testing territory
Section 6 already owns; this suite's entire reason to exist is the wiring between them.

## Running the suite

```bash
export DATABRICKS_HOST=https://your-workspace.cloud.databricks.com
export DATABRICKS_TOKEN=<uat-service-principal-token>
export DATABRICKS_CLUSTER_ID=<uat-cluster-id>
pytest tests_integration/ -v --timeout=1800
```

`--timeout=1800` (30 minutes) reflects reality this suite has to accept that Section 6's suite
never did -- a real pipeline run, a real DQ gate test with its own seeded batch, and a real
dashboard view check together take minutes, not the under-five-seconds Section 6's local suite
achieved. Slower is the correct trade here, for the different question this suite answers.

## Common mistakes

- **Running this suite against `dev` or `prod` "just to check."** `test_dq_check_failure_skips_transformation`
  deliberately seeds a batch designed to fail -- running it anywhere but a disposable, dedicated
  `uat` environment risks polluting real data or, worse, real production state with intentionally
  broken test fixtures.
- **Asserting exact row counts instead of a range or a relationship between two tables.** A fixed
  seed makes counts reproducible, but a Faker generator upgrade or a tuning change to injection
  rates would break every hardcoded literal count at once -- asserting `silver_count == bronze_count`
  (a relationship) survives that kind of change; asserting `count == 1204` (a literal) doesn't.
{: .important }

## What's next

A real, repeatable integration suite exists. Lecture 6 wires it into a CI/CD pipeline that runs it
automatically -- on every pull request against `uat`, before anything is ever allowed near `prod`.

<!-- prevnext:start -->

---

| [&larr; Previous: Planning and Designing Integration Testing Approach]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/planning-and-designing-integration-testing-approach/' | relative_url }}) | [Next: Develop and Trigger Your CI/CD Pipeline for Deployment and Integration Testing &rarr;]({{ '/02-stepright-capstone-project/08-stepright-integration-testing-packaging-and-deployment/develop-and-trigger-your-ci-cd-pipeline-for-deployment-and-integration-testing/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

