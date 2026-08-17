---
title: "Developing Bronze SDP Pipeline for File Sources"
parent: "StepRight - Ingestion Layer"
grand_parent: "StepRight Capstone Project"
nav_order: 4
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-sdp-pipeline-for-file-sources/
read_minutes: 5
---

# Developing Bronze SDP Pipeline for File Sources
{: .no_toc }

*Estimated read: 5 min*

Lecture 3 planned one table per landing subfolder, all sharing the same read shape. This lecture
builds that shape once, as a factory function, and generates all four tables from it.

## The factory function

```python
# transformations/bronze_files.py
import dlt as dp
from pyspark.sql.functions import col, current_timestamp

FILE_SOURCES = {
    "products": "json",
    "inventory": "json",
    "clickstream": "json",
    "fulfillment": "json",
}

def make_bronze_file_table(source_name: str, file_format: str):
    @dp.table(
        name=f"bronze_{source_name}",
        comment=f"Raw {source_name} files landed via Auto Loader, minimally transformed"
    )
    def _bronze_table():
        return (
            spark.readStream
            .format("cloudFiles")
            .option("cloudFiles.format", file_format)
            .option("cloudFiles.schemaLocation",
                    f"/Volumes/dev/step_right/checkpoints/bronze_{source_name}_schema/")
            .option("cloudFiles.schemaEvolutionMode", "rescue")
            .load(f"/Volumes/dev/step_right/landing/{source_name}/")
            .select(
                "*",
                col("_metadata.file_path").alias("_source_file"),
                current_timestamp().alias("_ingested_at"),
            )
        )
    return _bronze_table

for name, fmt in FILE_SOURCES.items():
    make_bronze_file_table(name, fmt)
```

The loop at the bottom is what actually registers all four tables with SDP -- calling
`make_bronze_file_table` four times with different closures over `source_name` and `file_format`
produces four distinct `@dp.table`-decorated functions, each pointed at its own landing subfolder
and its own dedicated schema location path. That per-source `schemaLocation` matters here for the
same reason it did in Part 1, Section 9: sharing one inference location across genuinely different
sources causes schema confusion between them.

## Why this is one file, not four

`bronze_products.py`, `bronze_inventory.py`, `bronze_clickstream.py`, and
`bronze_fulfillment.py` would each be nearly identical, differing only in a source name and file
format -- exactly the kind of duplication that makes a fifth source, added six months from now,
error-prone to add correctly. Adding a fifth file source to this project means adding one line to
the `FILE_SOURCES` dictionary, not writing and testing a new transformation file.

## Deploying alongside the CDC pipeline

This can live in the same `pipeline.yml` as Lecture 2's CDC tables (the `glob` include already
covers all of `transformations/**`), or as its own pipeline if you'd rather deploy and monitor
bronze CDC and bronze files independently:

```yaml
# pipeline.yml -- unchanged from Lecture 2 if reusing one pipeline
libraries:
  - glob:
      include: transformations/**
```

Splitting into two pipelines is a reasonable choice once a real team owns file-based sources and
CDC sources separately and wants independent deploy/monitor/alert boundaries -- for a project this
size, one pipeline covering both is simpler to run.

## Verifying all four landed

```sql
SELECT COUNT(*) FROM dev.step_right.bronze_products;
SELECT COUNT(*) FROM dev.step_right.bronze_inventory;
SELECT COUNT(*) FROM dev.step_right.bronze_clickstream;
SELECT COUNT(*) FROM dev.step_right.bronze_fulfillment;
```

Only `bronze_products` has real batch zero data at this point, since Section 1's loader notebook
only populated the `products/` landing subfolder -- the other three tables exist, are correctly
structured, and will populate the moment a batch adds files to their subfolders. An empty table
here isn't a bug; it's this pipeline correctly waiting for data that hasn't arrived yet.

With all seven bronze tables (three CDC, four file) landing correctly, Lectures 5 and 6 add the
quality tagging layer that decides what "correctly" actually means row by row.

<!-- prevnext:start -->

---

| [&larr; Previous: Planning and Designing Bronze Pipeline for File Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-file-sources/' | relative_url }}) | [Next: Design Source Data Quality Monitoring in Bronze Layer &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/design-source-data-quality-monitoring-in-bronze-layer/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

