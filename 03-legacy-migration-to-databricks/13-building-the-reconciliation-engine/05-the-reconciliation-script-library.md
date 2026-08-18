---
title: "The Reconciliation Script Library"
parent: "Building the Reconciliation Engine"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/the-reconciliation-script-library/
read_minutes: 3
---

# The Reconciliation Script Library
{: .no_toc }

*Estimated read: 3 min*

A migration with forty table pairs doesn't need forty reconciliation scripts -- it needs one engine and forty YAML files. That's the design goal for this final piece: take the hash-diff logic, the audit-table write, and the nightly-schedule pattern built earlier in this section and collapse them into a single, **pip-installable** library that every table pair drives through config rather than code.

Each table pair gets a small YAML definition describing what to compare and how, with nothing table-specific hardcoded into Python:

```yaml
table_pair: orders
source:
  connection: oracle_prod
  query: "SELECT order_id, customer_id, order_total, order_status FROM orders"
target:
  catalog: prod
  schema: silver
  table: orders
join_key: order_id
compare_columns: [customer_id, order_total, order_status]
tolerance:
  order_total: 0.01
```

The engine itself becomes a thin driver that reads one of these configs and calls the same `add_row_hash` and diff-join logic from the PySpark hash-diff lecture, parameterized entirely by what the YAML supplies:

```python
def run_reconciliation(config_path: str):
    cfg = load_config(config_path)
    source_df = read_source(cfg["source"])
    target_df = read_target(cfg["target"])
    diff = hash_diff(source_df, target_df, cfg["join_key"], cfg["compare_columns"])
    write_audit(diff, cfg["table_pair"], "recon.audit_log")
```

Package this as an installable module -- a `setup.py` or `pyproject.toml` with an entry point -- so any Lakeflow Job task in the migration can run `recon-engine --config orders.yaml` as a single library call rather than a copy-pasted notebook. This is the same shift **Lakebridge** made for schema and code translation earlier in this migration -- one reusable engine instead of one bespoke script per artifact -- applied here to verification instead of translation.

Two structural choices make the library scale cleanly to a large table inventory. First, **append-only** applies to both the delta output and the audit table -- no run ever overwrites a prior run's record, so the full history of every table pair's parity over the life of the migration stays queryable. Second, the config files themselves belong in version control alongside the migration codebase, so adding a forty-first table pair to nightly reconciliation is a one-file pull request, not a new script to write and review.

{: .important }
Resist the temptation to special-case one "difficult" table pair with a bespoke script instead of stretching the config schema to cover it -- the moment the library has two code paths, one general and one table-specific, the special case is the one nobody remembers to keep in sync when the engine's hash logic gets a bug fix. Extend the YAML schema instead.

The `tolerance` block deserves its own mention, since it's the schema field most likely to grow over time as the migration surfaces new numeric edge cases. Starting with a single per-column absolute tolerance, as in the `orders` example, covers most cases, but a mature library eventually needs a percentage-based tolerance for columns whose values span several orders of magnitude, and an explicit `exclude_columns` list for fields -- a `last_modified_at` timestamp, a surrogate key generated independently on each platform -- that will never match and shouldn't be compared at all. Treat the schema as versioned and additive: new fields get sensible defaults so a config written in week two of the migration still runs unmodified in week thirty.

Testing the library itself matters too, and cheaply: a small set of synthetic fixture tables -- a clean pair with no diffs, a pair with a known missing row, a pair with a known value mismatch -- run through the engine in CI on every change catches a regression in the hash or join logic before it silently corrupts a night's worth of real reconciliation runs. A verification tool that isn't itself under test is the one piece of this section's architecture where a bug is hardest to notice, because a broken reconciliation engine that reports false positives everywhere still *looks* like it's doing its job.

With the engine, the dashboard, the nightly schedule, and the portable library all in place, the section's quiz checks whether the architecture -- not just the individual pieces -- has landed.

<!-- prevnext:start -->

---

| [&larr; Previous: Start at Day 0, Not End of Project]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/start-at-day-0-not-end-of-project/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/13-building-the-reconciliation-engine/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

