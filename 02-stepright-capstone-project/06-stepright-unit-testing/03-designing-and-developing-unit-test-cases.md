---
title: "Designing and Developing Unit Test Cases"
parent: "StepRight - Unit Testing"
grand_parent: "StepRight Capstone Project"
nav_order: 3
permalink: /02-stepright-capstone-project/06-stepright-unit-testing/designing-and-developing-unit-test-cases/
read_minutes: 17
---

# Designing and Developing Unit Test Cases
{: .no_toc }

*Estimated read: 17 min*

`gold_logic.py` and `silver_logic.py` hold pure functions now. This lecture writes the actual
`pytest` suite against them -- a shared Spark fixture, a small library of test DataFrames, and a
full set of test cases covering `compute_daily_revenue`'s discount math, `dedupe_latest`'s window
logic, and `tag_quality`'s rule tagging.

## `conftest.py`: one local SparkSession, shared by every test

```python
# tests/conftest.py
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    session = (
        SparkSession.builder
        .master("local[2]")
        .appName("steprightproject-tests")
        .config("spark.sql.shuffle.partitions", "2")
        .getOrCreate()
    )
    yield session
    session.stop()
```

`scope="session"` matters for speed: starting a `SparkSession` costs real time (a few seconds),
and paying that cost once for the whole test run -- not once per test function -- is what keeps a
suite with dozens of tests finishing in seconds rather than minutes. `local[2]` runs Spark
in-process with two threads, no cluster, no Databricks workspace, no network call at all.
`spark.sql.shuffle.partitions` set down to `2` (from Spark's default of 200) avoids hundreds of
tiny, pointless shuffle partitions on test DataFrames that are a handful of rows -- a common cause
of a local test suite that runs far slower than the data volume would suggest.

## A small library of test data

Every test needs a DataFrame shaped like a real silver or bronze table, but with only the columns
and rows that specific test actually cares about -- a helper keeps that terse:

```python
# tests/conftest.py (continued)
@pytest.fixture
def order_items_df(spark):
    return spark.createDataFrame(
        [
            ("OI-1", "ORD-1", "PROD-1", 150.00, 1, 20),   # shoes, 20% off
            ("OI-2", "ORD-1", "PROD-2", 12.00, 2, 20),    # socks, 20% off
            ("OI-3", "ORD-1", "PROD-3", 60.00, 1, 0),     # sandals, no discount
        ],
        "order_item_id string, order_id string, product_id string, unit_price double, quantity int, discount_pct int",
    )

@pytest.fixture
def orders_df(spark):
    return spark.createDataFrame(
        [("ORD-1", "CUST-1", "2026-08-10", "DELIVERED")],
        "order_id string, customer_id string, order_date string, status string",
    )

@pytest.fixture
def customers_df(spark):
    return spark.createDataFrame(
        [("CUST-1", "West")],
        "customer_id string, region string",
    )

@pytest.fixture
def products_df(spark):
    return spark.createDataFrame(
        [("PROD-1", "Running"), ("PROD-2", "Running"), ("PROD-3", "Sandals")],
        "product_id string, category string",
    )
```

These four fixtures reconstruct the exact shoes/socks/sandals scenario Section 4, Lecture 1 worked
through by hand: a $150 shoe and a $12 sock both at 20% off, a $60 sandal with no discount at all,
one order, one customer, one day. Reusing a known worked example as the test's input means the
expected output is already derived and double-checked, not a new set of numbers invented just for
the test.

## Testing `compute_daily_revenue`'s discount allocation

```python
# tests/test_gold_logic.py
from gold_logic import compute_daily_revenue

def test_gross_revenue_by_line_item_sums_correctly(spark, order_items_df, orders_df, customers_df, products_df):
    result = compute_daily_revenue(order_items_df, orders_df, customers_df, products_df).collect()[0]
    assert result["gross_revenue"] == 234.00  # 150 + 24 + 60

def test_discount_allocated_proportionally_not_evenly(spark, order_items_df, orders_df, customers_df, products_df):
    result = compute_daily_revenue(order_items_df, orders_df, customers_df, products_df).collect()[0]
    # 20% of (150 + 24) = 34.80 -- NOT 34.80 / 3 split evenly across all three lines
    assert result["discount_amount"] == 34.80

def test_net_revenue_reconciles_against_gross_minus_discount(spark, order_items_df, orders_df, customers_df, products_df):
    result = compute_daily_revenue(order_items_df, orders_df, customers_df, products_df).collect()[0]
    assert result["net_revenue"] == result["gross_revenue"] - result["discount_amount"]

def test_undiscounted_line_contributes_zero_discount(spark, order_items_df, orders_df, customers_df, products_df):
    # Isolate just the sandals line (no discount) to prove 0% discount_pct means 0 discount_amount,
    # not some default or rounding artifact
    sandals_only = order_items_df.filter("product_id = 'PROD-3'")
    result = compute_daily_revenue(sandals_only, orders_df, customers_df, products_df).collect()[0]
    assert result["gross_revenue"] == 60.00
    assert result["discount_amount"] == 0.00
```

Four assertions, each isolating one specific claim Section 4, Lecture 1 made in prose: the total is
right, the discount is proportional rather than evenly split, net reconciles against gross minus
discount, and a 0% discount line contributes nothing. Splitting one worked example into four
narrow tests, rather than one test asserting the whole result row at once, means a future change
that breaks *only* the discount calculation fails exactly the test that says so -- not a single
generic `test_gold_daily_revenue` failure that gives no hint which part of the math broke.

## Testing the filters that have to happen before the join, not after

Section 4, Lecture 1's "Common mistakes" section warned about filtering `status != 'CANCELLED'`
*after* the `groupBy` instead of before it. `compute_daily_revenue` itself doesn't apply that
filter -- it's the caller's job, on the I/O side (Lecture 2) -- so the test that actually matters
here is at the *caller* boundary, not inside `compute_daily_revenue`:

```python
def test_cancelled_orders_excluded_before_calling_compute_daily_revenue(spark, order_items_df, customers_df, products_df):
    cancelled_order = spark.createDataFrame(
        [("ORD-1", "CUST-1", "2026-08-10", "CANCELLED")],
        "order_id string, customer_id string, order_date string, status string",
    )
    filtered_orders = cancelled_order.filter("status != 'CANCELLED'")
    result = compute_daily_revenue(order_items_df, filtered_orders, customers_df, products_df)
    assert result.count() == 0  # the join drops every line item once its order is filtered out
```

This test is really documenting a contract: `compute_daily_revenue` trusts its caller to have
already filtered cancelled orders and non-current SCD2 versions -- it's a deliberate design choice
from the refactor, not an oversight, and this test is what makes that contract explicit and
checked rather than an assumption living only in a comment.

## Grouping across multiple days, categories, and regions

Every test so far uses one order on one day -- enough to verify the math, not enough to verify the
`groupBy` itself keeps genuinely different groups separate rather than accidentally collapsing
them together:

```python
def test_separate_days_categories_and_regions_stay_in_separate_rows(spark, customers_df, products_df):
    order_items = spark.createDataFrame(
        [("OI-1", "ORD-1", "PROD-1", 100.00, 1, 0), ("OI-2", "ORD-2", "PROD-3", 50.00, 1, 0)],
        "order_item_id string, order_id string, product_id string, unit_price double, quantity int, discount_pct int",
    )
    orders = spark.createDataFrame(
        [("ORD-1", "CUST-1", "2026-08-10", "DELIVERED"), ("ORD-2", "CUST-2", "2026-08-11", "DELIVERED")],
        "order_id string, customer_id string, order_date string, status string",
    )
    customers = spark.createDataFrame(
        [("CUST-1", "West"), ("CUST-2", "East")],
        "customer_id string, region string",
    )
    result = compute_daily_revenue(order_items, orders, customers, products_df).collect()
    assert len(result) == 2  # different date AND different region AND different category -- two distinct groups
```

`PROD-1` (Running) and `PROD-3` (Sandals) already differ in category from the earlier fixtures;
combined with different order dates and different customer regions here, this confirms the
`groupBy(revenue_date, category, region)` key genuinely separates rows that differ on any one of
those three columns, not just the obviously different ones.

## Floating-point comparisons: why `pytest.approx` matters for currency math

Every assertion so far compares against a value that happens to divide evenly (`34.80`, `234.00`).
Real discount math doesn't always land on a clean number -- a `discount_pct` of 15 against a
`$19.99` item produces `2.9985`, and Python's binary floating-point arithmetic can leave a result
like `2.9984999999999995` instead of the mathematically exact value. A direct `==` comparison
against that kind of result is a classic source of a test that fails intermittently for reasons
that have nothing to do with whether the logic is actually correct:

```python
import pytest

def test_discount_with_non_round_percentage_uses_approx_not_exact_equality(spark, orders_df, customers_df):
    order_items = spark.createDataFrame(
        [("OI-1", "ORD-1", "PROD-1", 19.99, 1, 15)],
        "order_item_id string, order_id string, product_id string, unit_price double, quantity int, discount_pct int",
    )
    products = spark.createDataFrame([("PROD-1", "Running")], "product_id string, category string")
    result = compute_daily_revenue(order_items, orders_df, customers_df, products).collect()[0]
    assert result["discount_amount"] == pytest.approx(2.9985, abs=0.01)
```

`pytest.approx(..., abs=0.01)` asserts "within one cent," which is the right tolerance for a
currency figure -- precise enough to catch a genuine calculation bug, loose enough to never
false-fail on floating-point representation noise. Every earlier test in this lecture used exact
`==` safely only because its inputs happened to produce clean decimal results; a real test suite
covering realistic discount percentages needs `pytest.approx` as the default, not the exception.

## Testing `tag_business_rules`: critical failures vs. warnings

Section 3's business-rule tagging (`tag_business_rules`) is a close cousin of `tag_quality`, but
splits failures into two independent buckets -- `_dq_critical_failed` (hold the row back) and
`_dq_warnings` (report only, let the row through) -- which needs its own coverage:

```python
# tests/test_dq_helpers.py (continued)
from dq_helpers import tag_business_rules
from pyspark.sql.functions import col

def test_critical_failure_sets_dq_critical_failed_true(spark):
    df = spark.createDataFrame([(-5.0,)], "list_price double")
    critical = {"negative_price": col("list_price") < 0}
    result = tag_business_rules(df, critical_checks=critical, warning_checks={}).collect()[0]
    assert result["_dq_critical_failed"] is True

def test_warning_only_does_not_set_dq_critical_failed(spark):
    df = spark.createDataFrame([("UNKNOWN_CATEGORY",)], "category string")
    warnings = {"unknown_category": col("category") == "UNKNOWN_CATEGORY"}
    result = tag_business_rules(df, critical_checks={}, warning_checks=warnings).collect()[0]
    assert result["_dq_critical_failed"] is False
    assert result["_dq_warnings"] == ["unknown_category"]

def test_empty_critical_checks_dict_never_fails_any_row(spark):
    # silver_customers and silver_products both call tag_business_rules with critical_checks={}
    df = spark.createDataFrame([(None,)], "region string")
    critical = {}
    result = tag_business_rules(df, critical_checks=critical, warning_checks={}).collect()[0]
    assert result["_dq_critical_failed"] is False
```

That third test is the one worth lingering on: it's asserting a *negative* -- that an empty
`critical_checks` dictionary can never fail a row, no matter what the row contains. That's exactly
the property `silver_customers` and `silver_products` depend on (Section 3, Lecture 3 and Section
3, Lecture 5 both pass `critical_checks={}` deliberately), and it's the kind of implicit contract
that's easy to break by accident in a future edit to `tag_business_rules` without a test calling it
out explicitly.

## Testing `dedupe_latest`

```python
# tests/test_silver_logic.py
from silver_logic import dedupe_latest

def test_keeps_only_the_most_recent_row_per_partition(spark):
    df = spark.createDataFrame(
        [
            ("PROD-1", "2026-08-09 10:00:00", 49.99),
            ("PROD-1", "2026-08-10 14:00:00", 44.99),  # newer -- this one should win
            ("PROD-2", "2026-08-10 09:00:00", 29.99),
        ],
        "product_id string, _ingested_at string, list_price double",
    )
    result = dedupe_latest(df, "product_id", "_ingested_at").collect()
    prices = {row["product_id"]: row["list_price"] for row in result}
    assert len(result) == 2
    assert prices["PROD-1"] == 44.99

def test_single_row_per_partition_passes_through_unchanged(spark):
    df = spark.createDataFrame([("PROD-1", "2026-08-10 09:00:00", 29.99)], "product_id string, _ingested_at string, list_price double")
    result = dedupe_latest(df, "product_id", "_ingested_at").collect()
    assert len(result) == 1
    assert result[0]["list_price"] == 29.99
```

The first test is the one that actually matters -- proving "latest wins" picks the *newer*
timestamp's price, not just any row. A test that only checked `count() == 2` without checking
*which* price survived would pass even if `dedupe_latest` kept the oldest row by mistake.

## Testing `tag_quality`

```python
# tests/test_dq_helpers.py
from dq_helpers import tag_quality
from pyspark.sql.functions import col

def test_row_passing_all_checks_is_marked_valid(spark):
    df = spark.createDataFrame([("ORD-1", "CUST-1")], "order_id string, customer_id string")
    checks = {"missing_order_id": col("order_id").isNull()}
    result = tag_quality(df, checks).collect()[0]
    assert result["_dq_valid"] is True
    assert result["_dq_failed_rules"] == []

def test_row_failing_a_check_is_tagged_with_the_rule_name(spark):
    df = spark.createDataFrame([(None, "CUST-1")], "order_id string, customer_id string")
    checks = {"missing_order_id": col("order_id").isNull()}
    result = tag_quality(df, checks).collect()[0]
    assert result["_dq_valid"] is False
    assert result["_dq_failed_rules"] == ["missing_order_id"]

def test_row_can_fail_multiple_checks_at_once(spark):
    df = spark.createDataFrame([(None, None)], "order_id string, customer_id string")
    checks = {
        "missing_order_id": col("order_id").isNull(),
        "missing_customer_id": col("customer_id").isNull(),
    }
    result = tag_quality(df, checks).collect()[0]
    assert sorted(result["_dq_failed_rules"]) == ["missing_customer_id", "missing_order_id"]
```

`tag_quality` needed no refactoring, but that doesn't mean it needed no tests -- it's arguably the
single most-reused piece of logic in the whole project, called by every quality-tagged bronze
table across Section 2. A bug here would silently affect all seven sources at once, which makes
covering it directly, with its own dedicated test file, worth the few extra lines.

## Comparing full DataFrames, not just one collected row

Every test above pulls a single row with `.collect()[0]` for readability, but a test asserting
*multiple* output rows needs actual DataFrame comparison, not row-by-row indexing that assumes a
particular order:

```python
def assert_dataframes_equal(actual, expected):
    actual_rows = sorted(actual.collect(), key=lambda r: tuple(r.asDict().values().__str__()))
    expected_rows = sorted(expected.collect(), key=lambda r: tuple(r.asDict().values().__str__()))
    assert actual_rows == expected_rows
```

Sorting both sides before comparing is what makes this safe against Spark's own row ordering, which
is never guaranteed across a `groupBy` -- a naive `actual.collect() == expected.collect()` can fail
on two DataFrames holding the exact same rows in a different order, a flaky-test trap worth
avoiding from the first test that needs multi-row comparison:

```python
def test_two_distinct_groups_match_expected_dataframe_exactly(spark, customers_df):
    order_items = spark.createDataFrame(
        [("OI-1", "ORD-1", "PROD-1", 100.00, 1, 0), ("OI-2", "ORD-2", "PROD-3", 50.00, 1, 0)],
        "order_item_id string, order_id string, product_id string, unit_price double, quantity int, discount_pct int",
    )
    orders = spark.createDataFrame(
        [("ORD-1", "CUST-1", "2026-08-10", "DELIVERED"), ("ORD-2", "CUST-1", "2026-08-11", "DELIVERED")],
        "order_id string, customer_id string, order_date string, status string",
    )
    products = spark.createDataFrame(
        [("PROD-1", "Running"), ("PROD-3", "Sandals")], "product_id string, category string"
    )
    expected = spark.createDataFrame(
        [
            ("2026-08-10", "Running", "West", 100.00, 0.00, 100.00, 1),
            ("2026-08-11", "Sandals", "West", 50.00, 0.00, 50.00, 1),
        ],
        "revenue_date string, category string, region string, gross_revenue double, discount_amount double, net_revenue double, line_item_count long",
    )
    actual = compute_daily_revenue(order_items, orders, customers_df, products)
    assert_dataframes_equal(actual, expected)
```

Writing out `expected` as a literal DataFrame, column for column, is more verbose than asserting a
single field -- but it's the only way to verify the *entire* output shape at once, including that
no extra or missing rows snuck in, which single-field assertions structurally can't catch.

## Making `gold_logic` and friends importable from `tests/`

None of the tests above work until `pytest` can actually resolve `from gold_logic import
compute_daily_revenue` -- `transformations/` needs to be on the Python path when tests run, which
`conftest.py` handles once, for every test file in the folder:

```python
# tests/conftest.py (top of file, before the fixtures)
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "transformations"))
```

`conftest.py` runs once per test session before any test file imports anything, which makes it the
right place for this path setup -- doing it inside an individual test file would only fix that one
file's imports, not the whole suite's.

## Edge case: an empty input DataFrame

A day with genuinely zero orders (a plausible Monday-morning-before-opening scenario, or simply the
first day this pipeline ever ran) shouldn't crash `compute_daily_revenue` -- it should just return
an empty result:

```python
def test_empty_order_items_returns_empty_result_not_an_error(spark, orders_df, customers_df, products_df):
    empty_order_items = spark.createDataFrame(
        [], "order_item_id string, order_id string, product_id string, unit_price double, quantity int, discount_pct int"
    )
    result = compute_daily_revenue(empty_order_items, orders_df, customers_df, products_df)
    assert result.count() == 0
```

An empty-input test costs almost nothing to write and catches a real, if uncommon, class of bug:
an aggregation or join written in a way that throws on zero rows instead of degrading gracefully to
an empty result, something manual spot-checking against real StepRight data would likely never
happen to exercise, since real data essentially never has zero rows.

## Common mistakes

- **Asserting exact equality on a computed currency value.** Any test comparing a `discount_amount`
  or `net_revenue` result against a literal float with `==` is one non-round percentage away from
  an intermittent, confusing failure -- reach for `pytest.approx` by default on any monetary
  calculation, not only once a test happens to fail.
- **Building enormous fixture DataFrames "to be realistic."** A ten-thousand-row test fixture
  doesn't test the logic any more thoroughly than three carefully chosen rows do -- it only makes
  the test slower to run and harder to read, since the actual assertion gets buried under
  irrelevant scale. Keep test data as small as the specific behavior being verified requires.
- **One test function asserting five unrelated things.** A test that checks gross revenue,
  discount, net revenue, and row count all in one function makes it unclear, on a failure, which
  claim actually broke -- the narrow, single-assertion tests throughout this lecture are the
  deliberate alternative.
{: .important }

## What's next

Every test in this lecture proves one function's logic in isolation. Lecture 4 rounds out the
suite with `dq_check`'s threshold logic, runs the whole thing with `pytest`, and looks at what a
real failing run looks like in the test output.

<!-- prevnext:start -->

---

| [&larr; Previous: Refactoring Your Code and Getting Ready for Unit Testing]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/refactoring-your-code-and-getting-ready-for-unit-testing/' | relative_url }}) | [Next: Code and Run all Unit Test Case &rarr;]({{ '/02-stepright-capstone-project/06-stepright-unit-testing/code-and-run-all-unit-test-case/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

