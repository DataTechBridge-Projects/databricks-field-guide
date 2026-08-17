---
title: "StepRight - Ingestion Layer"
parent: "StepRight Capstone Project"
nav_order: 2
has_children: true
permalink: /02-stepright-capstone-project/02-stepright-ingestion-layer/
---

# StepRight - Ingestion Layer

With `dev.step_right` provisioned and batch zero seeded, this section builds the first layer that
actually runs: bronze. StepRight's two source shapes -- Lakeflow Connect's CDC feed and Auto
Loader's file drops -- get separate pipelines because they fail differently and need different
schema discipline, and both feed a shared quarantine pattern so a structurally broken row gets
flagged and set aside instead of silently corrupting everything downstream in silver.

```mermaid
flowchart TD
    subgraph CDC["dev.raw_cdc"]
        C1[customers_changefeed]
        C2[orders_changefeed]
        C3[order_items_changefeed]
    end
    subgraph Files["landing volume"]
        F1[products/]
        F2[inventory/]
        F3[clickstream/]
        F4[fulfillment/]
    end
    C1 & C2 & C3 -->|fixed schema, Lecture 1-2| BC[Bronze CDC Tables]
    F1 & F2 & F3 & F4 -->|Auto Loader, rescue mode, Lecture 3-4| BF[Bronze File Tables]
    BC --> BQ[Bronze Quality Tagging<br/>Lecture 5-6]
    BF --> BQ
    BQ -->|valid rows| Valid[(bronze_*_valid)]
    BQ -->|failed rows| Quar[(bronze_*_quarantine)]
    Valid -.->|read by Section 3 silver| Silver[(Silver Layer)]
```

## Lectures

| # | Lecture | Est. Read |
|---|---------|-----------|
| 1 | [Planning and Designing Bronze Pipeline for CDC Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-cdc-sources/' | relative_url }}) | 7 min read |
| 2 | [Developing Bronze Pipeline for CDC Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-pipeline-for-cdc-sources/' | relative_url }}) | 19 min read |
| 3 | [Planning and Designing Bronze Pipeline for File Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-file-sources/' | relative_url }}) | 3 min read |
| 4 | [Developing Bronze SDP Pipeline for File Sources]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-bronze-sdp-pipeline-for-file-sources/' | relative_url }}) | 5 min read |
| 5 | [Design Source Data Quality Monitoring in Bronze Layer]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/design-source-data-quality-monitoring-in-bronze-layer/' | relative_url }}) | 9 min read |
| 6 | [Developing Quarantine Pattern in Bronze Layer]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/developing-quarantine-pattern-in-bronze-layer/' | relative_url }}) | 13 min read |

<!-- prevnext:start -->

---

| [&larr; Previous: Download Project Source Code]({{ '/02-stepright-capstone-project/01-stepright-project-foundation/download-project-source-code/' | relative_url }}) | [Next: Planning and Designing Bronze Pipeline for CDC Sources &rarr;]({{ '/02-stepright-capstone-project/02-stepright-ingestion-layer/planning-and-designing-bronze-pipeline-for-cdc-sources/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

