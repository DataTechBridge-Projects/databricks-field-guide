---
title: "Code and Run all Unit Test Case"
parent: "StepRight - Unit Testing"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/06-stepright-unit-testing/code-and-run-all-unit-test-case/
read_minutes: 12
---

# Code and Run all Unit Test Case
{: .no_toc }

*Estimated read: 12 min*

Three test files exist, covering `gold_logic`, `silver_logic`, and `dq_helpers`. One piece of
real logic is still untested: Section 5's `dq_check.py` threshold decision -- and it has the same
problem `gold_daily_revenue` had before Lecture 2's refactor. This lecture extracts it, tests it,
and runs the complete suite for real.

## `dq_check.py` mixes a SQL query with a decision -- again

Section 5, Lecture 2's `dq_check.py` computes each source's quarantine rate and decides pass/fail
inline, in the same function that calls `spark.sql(...)`:

```python
# Before -- Section 5, Lecture 2
for source in SOURCES:
    valid = spark.sql(f"SELECT count(*) FROM ... WHERE date(_ingested_at) = '{args.run_date}'").collect()[0][0]
    quarantined = spark.sql(f"SELECT count(*) FROM ... WHERE date(_ingested_at) = '{args.run_date}'").collect()[0][0]
    total = valid + quarantined
    rate = quarantined / total if total > 0 else 0.0
    if rate > QUARANTINE_THRESHOLD:
        failures.append(f"{source}: {rate:.1%} exceeds {QUARANTINE_THRESHOLD:.0%} threshold")
```

The decision logic -- given counts, is this a failure -- doesn't need Spark at all. Pulling it out
means it can be tested with plain Python integers, no `SparkSession` fixture required:

```python
# transformations/dq_logic.py
def quarantine_rate(valid_count: int, quarantine_count: int) -> float:
    total = valid_count + quarantine_count
    return quarantine_count / total if total > 0 else 0.0

def evaluate_thresholds(counts: dict, threshold: float) -> list[str]:
    """counts: {source_name: (valid_count, quarantine_count)}. Returns a list of failure messages."""
    failures = []
    for source, (valid, quarantined) in counts.items():
        rate = quarantine_rate(valid, quarantined)
        if rate > threshold:
            failures.append(f"{source}: {rate:.1%} exceeds {threshold:.0%} threshold")
    return failures
```

```python
# transformations/dq_check.py -- now a thin wrapper, same shape as Lecture 2's pipeline refactors
import argparse
import sys
from databricks.sdk.runtime import spark
from dq_logic import evaluate_thresholds

SOURCES = ["customers", "orders", "order_items", "products", "inventory", "clickstream", "fulfillment"]
QUARANTINE_THRESHOLD = 0.05

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--run_date", required=True)
    args = parser.parse_args()

    counts = {}
    for source in SOURCES:
        valid = spark.sql(f"SELECT count(*) FROM dev.step_right.bronze_{source}_valid WHERE date(_ingested_at) = '{args.run_date}'").collect()[0][0]
        quarantined = spark.sql(f"SELECT count(*) FROM dev.step_right.bronze_{source}_quarantine WHERE date(_ingested_at) = '{args.run_date}'").collect()[0][0]
        counts[source] = (valid, quarantined)

    failures = evaluate_thresholds(counts, QUARANTINE_THRESHOLD)
    if failures:
        print("DQ CHECK FAILED:")
        for f in failures:
            print(f"  - {f}")
        sys.exit(1)
    print("DQ CHECK PASSED")

if __name__ == "__main__":
    main()
```

Notice `dq_logic.py` needs no `SparkSession` fixture at all -- `quarantine_rate` and
`evaluate_thresholds` operate on plain Python `int`, `float`, `dict`, and `list`, not DataFrames.
Not every extraction in this project produces a *Spark* unit test; some produce an even simpler
plain-Python one, which is a good sign, not a gap -- the less machinery a test needs to set up, the
faster and more reliable it runs.

## Testing the threshold decision

```python
# tests/test_dq_logic.py
from dq_logic import quarantine_rate, evaluate_thresholds

def test_quarantine_rate_basic_division():
    assert quarantine_rate(valid_count=95, quarantine_count=5) == 0.05

def test_quarantine_rate_with_zero_total_returns_zero_not_a_division_error():
    assert quarantine_rate(valid_count=0, quarantine_count=0) == 0.0

def test_rate_exactly_at_threshold_does_not_fail():
    # Section 5, Lecture 2 uses `> threshold`, not `>=` -- exactly 5% should pass
    failures = evaluate_thresholds({"orders": (95, 5)}, threshold=0.05)
    assert failures == []

def test_rate_just_over_threshold_fails():
    failures = evaluate_thresholds({"orders": (94, 6)}, threshold=0.05)
    assert len(failures) == 1
    assert "orders" in failures[0]

def test_only_the_offending_source_appears_in_failures():
    counts = {"orders": (99, 1), "clickstream": (60, 40)}
    failures = evaluate_thresholds(counts, threshold=0.05)
    assert len(failures) == 1
    assert "clickstream" in failures[0]
    assert "orders" not in failures[0]
```

`test_rate_exactly_at_threshold_does_not_fail` is the one worth pausing on -- it's testing the
*boundary* condition of a `>` comparison, exactly the kind of off-by-one mistake (`>` vs. `>=`)
that's easy to introduce silently in a future edit and that a test asserting only "clearly under"
and "clearly over" values would never catch.

## One more edge case: `dedupe_latest`'s tie-breaking gap

Section 3, Lecture 5's own "Verifying the result" section flagged a real, known gap in
`dedupe_latest`: two rows sharing the exact same `_ingested_at` timestamp break ties arbitrarily,
since `row_number()` needs some deterministic order and timestamp alone doesn't guarantee one.
Rather than leave that as a comment, this is worth a test that documents the *current* behavior
honestly, so a future fix has something concrete to change:

```python
def test_tied_timestamps_pick_one_row_but_which_one_is_not_guaranteed(spark):
    df = spark.createDataFrame(
        [("PROD-1", "2026-08-10 09:00:00", 29.99), ("PROD-1", "2026-08-10 09:00:00", 34.99)],
        "product_id string, _ingested_at string, list_price double",
    )
    result = dedupe_latest(df, "product_id", "_ingested_at").collect()
    # Known gap (Section 3, Lecture 5): exactly one row survives, but which one is unspecified.
    # This test locks in "exactly one row," not "the right one" -- a real tie-break fix would need
    # a secondary sort column, tracked as a follow-up rather than silently assumed correct here.
    assert len(result) == 1
```

A test like this isn't asserting the code is bug-free -- it's asserting the *known* limitation
stays exactly as documented, so a future change to `dedupe_latest` that accidentally makes ties
non-deterministic in a *new* way (returning zero or two rows instead of one) still gets caught,
even though the underlying "which one wins" ambiguity remains open.

## Configuring pytest

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --strict-markers
```

`testpaths = tests` means a bare `pytest` command, run from the repo root, finds everything without
needing the `tests/` argument spelled out every time -- worth having in the repo from the first
test file, not added later once the suite has grown large enough to make omitting it annoying.

## Measuring coverage

```bash
pip install pytest-cov
pytest tests/ --cov=transformations --cov-report=term-missing
```

```text
Name                         Stmts   Miss  Cover   Missing
----------------------------------------------------------
transformations/dq_helpers.py    18      0   100%
transformations/dq_logic.py       9      0   100%
transformations/gold_logic.py    12      0   100%
transformations/silver_logic.py   7      0   100%
----------------------------------------------------------
TOTAL                            46      0   100%
```

100% coverage here is a meaningful number specifically *because* the scope is narrow -- these four
files are exactly the pure logic Sections 2 through 5 extracted for testability, not the entire
`transformations/` folder. The `@dp.table`-decorated wrapper functions correctly show up as
untested by this report, and that's expected: their I/O is what Section 8's integration tests
exist to cover, not this suite. A coverage report is only useful read against what it was actually
scoped to check.

## The full `tests/` directory

```text
tests/
├── conftest.py           # spark fixture, path setup, shared DataFrame fixtures
├── test_dq_helpers.py    # tag_quality, tag_business_rules
├── test_dq_logic.py      # quarantine_rate, evaluate_thresholds (this lecture)
├── test_gold_logic.py    # compute_daily_revenue
└── test_silver_logic.py  # dedupe_latest
```

Five files, roughly twenty test functions total across the section -- small individually, but
together covering every piece of pure business logic this capstone has built since Section 2.

## Running one test at a time during development

The full suite in under five seconds is fast enough to run on every save, but a single failing
test during active development doesn't need the other nineteen re-run alongside it:

```bash
pytest tests/test_gold_logic.py::test_discount_allocated_proportionally_not_evenly -v
```

Scoping to one file, or one specific test function within it, is the normal day-to-day rhythm while
writing or fixing a test -- run the whole suite before committing, but iterate against just the one
test actually being worked on in between.

## Local PySpark vs. Databricks Connect, one more time

| | Local PySpark (this section) | Databricks Connect (Section 8) |
|---|---|---|
| Startup cost | ~2 seconds, once per session | Cluster start: minutes, or reuse an already-running one |
| Data | In-memory fixtures, defined in the test itself | Real or test-environment Unity Catalog tables |
| Runs in CI without a workspace | Yes | No -- needs live workspace credentials |
| What it proves | The function's logic is correct for a known input | The deployed pipeline behaves correctly against real infrastructure |

Neither column is strictly "better" -- they answer different questions, which is why Section 8
adds Databricks Connect-based integration tests rather than replacing this section's suite with
them. A change that breaks `compute_daily_revenue`'s math fails here, in seconds, on a laptop with
no Databricks access at all; a change that breaks how `steprightproject-silver-gold` is actually
wired together in Unity Catalog fails in Section 8's suite instead, since no local fixture could
ever catch a deployment-configuration mistake this section's tests were never scoped to see.

## Running the suite

```bash
pip install -r requirements-test.txt   # pytest, pyspark
pytest tests/ -v
```

```text
tests/test_dq_helpers.py::test_row_passing_all_checks_is_marked_valid PASSED
tests/test_dq_helpers.py::test_row_failing_a_check_is_tagged_with_the_rule_name PASSED
tests/test_dq_helpers.py::test_row_can_fail_multiple_checks_at_once PASSED
tests/test_dq_helpers.py::test_critical_failure_sets_dq_critical_failed_true PASSED
tests/test_dq_helpers.py::test_warning_only_does_not_set_dq_critical_failed PASSED
tests/test_dq_logic.py::test_quarantine_rate_basic_division PASSED
tests/test_dq_logic.py::test_rate_exactly_at_threshold_does_not_fail PASSED
tests/test_dq_logic.py::test_rate_just_over_threshold_fails PASSED
tests/test_gold_logic.py::test_gross_revenue_by_line_item_sums_correctly PASSED
tests/test_gold_logic.py::test_discount_allocated_proportionally_not_evenly PASSED
tests/test_gold_logic.py::test_net_revenue_reconciles_against_gross_minus_discount PASSED
tests/test_silver_logic.py::test_keeps_only_the_most_recent_row_per_partition PASSED
============================== 20 passed in 4.87s ==============================
```

Under five seconds for the whole suite -- the direct payoff of local PySpark over a live cluster:
no cluster startup, no network round trip, nothing but the two-second cost of starting Spark
in-process, paid once thanks to `conftest.py`'s session-scoped fixture.

## What a genuine failure looks like

Suppose a future edit accidentally changes `compute_daily_revenue`'s discount formula to round
`discount_pct` before multiplying -- the exact bug Lecture 1's motivating scenario described:

```text
tests/test_gold_logic.py::test_discount_allocated_proportionally_not_evenly FAILED

    def test_discount_allocated_proportionally_not_evenly(spark, order_items_df, orders_df, customers_df, products_df):
        result = compute_daily_revenue(order_items_df, orders_df, customers_df, products_df).collect()[0]
>       assert result["discount_amount"] == 34.80
E       assert 35.00 == 34.80

tests/test_gold_logic.py:15: AssertionError
========================== 1 failed, 19 passed in 4.91s ==========================
```

This is the whole point of Section 6: a discount-formula bug like this one gets caught here,
locally, in under five seconds, with a stack trace pointing at the exact function and the exact
expected-vs-actual numbers -- not weeks later, in a finance dashboard that quietly stopped
reconciling, with `dq_check` still reporting a perfectly healthy 1-2% quarantine rate every single
night.

## Common mistakes

- **Writing `dq_logic.py`'s tests against Spark DataFrames when plain Python types would do.**
  `quarantine_rate` and `evaluate_thresholds` never touch a DataFrame -- wrapping their test inputs
  in `spark.createDataFrame` anyway would only add unnecessary Spark session overhead to tests that
  don't need it.
- **Letting the test suite go stale as pipelines change.** A test suite is only as valuable as its
  last run -- if Section 7 or Section 8 later changes `compute_daily_revenue`'s signature without
  updating `test_gold_logic.py` to match, a suite nobody runs anymore, or that's been silently
  skipped, provides exactly zero of the protection this section spent four lectures building.
{: .important }

## Why every test name in this section is a full sentence

`test_discount_allocated_proportionally_not_evenly`, not `test_discount_1`. Every test function
across all four lectures follows the same convention: a name long enough to state the specific
claim being verified, not just which function is under test. The payoff shows up exactly in the
failure output above -- `pytest`'s `-v` flag prints the test name itself as part of the result
line, which means a failing test's name alone, before reading a single line of the traceback,
already says *what broke*: "discount allocated proportionally, not evenly" failing is immediately
more informative than "test_discount_1" failing, especially months from now when whoever's reading
the CI output didn't write the test and has no memory of what "case 1" was supposed to mean.

## Section wrap-up

Twenty-odd tests, five files, one Spark fixture, and a five-second run time -- StepRight's
transformation logic is now provably correct against known inputs, independent of whatever real
data happens to be sitting in `dev.step_right` on a given day. Section 7 turns to a different
concern: tracking data quality trends *over time*, the rolling-average, threshold-drift monitoring
this section's `dq_check` deliberately left for later.

<!-- prevnext:start -->

---

| [&larr; Previous: Designing and Developing Unit Test Cases]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/designing-and-developing-unit-test-cases/' | relative_url }}) | [Next: StepRight - Data Quality Monitoring &rarr;]({{ '/02-stepright-capstone-project/07-stepright-data-quality-monitoring/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

