---
title: "The Data Ferry: Schedule, Capacity, Manifest"
parent: "The Ingestion Decision Tree"
grand_parent: "Legacy Migration to Databricks"
nav_order: 1
permalink: /03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/the-data-ferry-schedule-capacity-manifest/
read_minutes: 2
---

# The Data Ferry: Schedule, Capacity, Manifest
{: .no_toc }

*Estimated read: 2 min*

Every legacy nightly-batch pipeline you've ever operated implicitly agreed to a contract with its
source system, even if nobody wrote it down: a **schedule** (when the load runs and how long it's
allowed to take), a **capacity** (how much data or throughput the job is sized for), and a
**manifest** (a record of exactly which files or rows were expected, so a partial or missing feed
is detected rather than silently treated as "complete"). Think of ingestion as a **data ferry**: it
doesn't matter whether the ferry runs on a fixed timetable or continuously -- what matters is that
it never leaves the dock overloaded, and it never claims to have delivered cargo it doesn't
actually have manifested.

## The three terms of the contract

- **Schedule.** In the legacy world this was a cron entry or a `DBMS_SCHEDULER` job tied to a
  maintenance window. On Databricks it becomes a Lakeflow Jobs trigger -- either a cron-style
  schedule for batch, or a continuous/triggered Structured Streaming job for near-real-time. The
  schedule question doesn't disappear; it just moves from "when does the window open" to "how
  often should this pipeline check for new data."
- **Capacity.** The legacy job was sized against a fixed on-prem server -- a known number of cores,
  a known amount of memory, a known number of JDBC connections the source database could tolerate
  before it started paging the DBA. On Databricks, capacity is a cluster policy and an
  **autoscaling** range, but the underlying constraint doesn't vanish: the *source* system (an
  Oracle instance, a Teradata BTEQ export) still has a finite number of connections and I/O
  bandwidth it can give up before production OLTP traffic suffers.
- **Manifest.** The legacy job's manifest was often a control file: a count of expected records, a
  checksum, or a `.done` marker file dropped only after the full extract succeeded. Auto Loader's
  file-notification and checkpoint state serve the same purpose for file-based ingestion, and a
  well-designed JDBC extract still needs an explicit row-count or watermark check -- the mechanism
  changes, but skipping this term of the contract is exactly how "the load succeeded" and "the load
  actually got all the data" quietly diverge.

## The fail-fast guard

The instinct in a rushed migration is to port the *happy path* of a legacy job and skip the
contract enforcement, because it "worked fine for years." That's backwards: the guard code is
usually the only thing standing between a missing upstream file and a silently short gold-layer
report. A minimal version of that guard, adapted for a Lakeflow Jobs task:

```python
expected_files = get_manifest_for_batch(batch_date)
actual_files = dbutils.fs.ls(f"/Volumes/bronze/orders/incoming/{batch_date}")

missing = set(expected_files) - {f.name for f in actual_files}
if missing:
    raise ValueError(f"Ingestion contract violated: missing files {missing}")
```

{: .important }
> A pipeline that ingests whatever files happen to be present, with no manifest check, has traded
> an explicit failure for a silent one. On a legacy warehouse that failure showed up as a
> mismatched row count someone eventually noticed; on Databricks it shows up the same way, unless
> you enforce the contract at the door.

The rest of this section works through the choices that fill in *how* the schedule, capacity, and
manifest terms get implemented -- batch vs. streaming, JDBC vs. Auto Loader, build vs. buy -- but
every one of those choices still has to answer to this same three-term contract.

<!-- prevnext:start -->

---

| [&larr; Previous: The Ingestion Decision Tree]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/' | relative_url }}) | [Next: Batch vs Streaming, Push vs Pull, Schema-on-Read vs Write &rarr;]({{ '/03-legacy-migration-to-databricks/10-the-ingestion-decision-tree/batch-vs-streaming-push-vs-pull-schema-on-read-vs-write/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

