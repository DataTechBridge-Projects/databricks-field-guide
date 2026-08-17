---
title: "Databricks Platform Access - Paid vs Free"
parent: "Introduction"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/02-introduction/databricks-platform-access-paid-vs-free/
read_minutes: 4
---

# Databricks Platform Access - Paid vs Free
{: .no_toc }

*Estimated read: 4 min*

Before you create anything, decide which access path you actually need -- the next two sections
diverge based on this choice, and there's no reason to set up a paid AWS-billed workspace if
Free Edition covers what you're trying to learn.

## Databricks Free Edition

**Databricks Free Edition** (the successor to the older "Community Edition") gives you a real,
serverless Databricks workspace at no cost, with no credit card required. It's not a crippled demo
-- you get notebooks, Delta Lake tables, Unity Catalog, and enough compute to work through
essentially all of Part 1 and most of Part 2 of this guide. See the official
[Databricks Free Edition page](https://www.databricks.com/learn/free-edition) for the current
feature set and signup flow, covered hands-on in the next lecture.

**Who should use it:** anyone working through Part 1 and Part 2 without an existing AWS account,
or anyone who just wants to try Databricks before deciding whether to set up billing at all.

## Paid access via AWS

**Paid access** means a Databricks account tied to real billing -- in this guide's case, through
the **AWS Marketplace**, which lets you subscribe to Databricks the same way you'd subscribe to
any other AWS Marketplace product, with usage billed alongside the rest of your AWS spend rather
than through a separate invoice relationship. See
[Databricks pricing](https://www.databricks.com/product/pricing) for the current pay-as-you-go and
committed-use models; note that AWS Marketplace billing sits on top of this as the payment
mechanism, not a separate pricing tier.

**Who needs this:** anyone following Section 3's AWS account and workspace walkthrough, or anyone
specifically preparing for [Part 3's migration content]({{ '/03-legacy-migration-to-databricks/' | relative_url }}), which
assumes a real workspace connected to your own cloud account with IAM roles and S3 buckets you
control -- something Free Edition's fully-managed serverless model doesn't expose.

## Practical guidance for this guide

| Path | Section 3 required? | Covers |
|---|---|---|
| Free Edition | No -- skip to Section 4 | Parts 1-2 (Fundamentals, StepRight capstone) |
| Paid, AWS Marketplace | Yes | Everything, including Part 3's migration content |

If you're not yet sure which you'll need, **start with Free Edition**. Nothing about the paid setup
in Section 3 is a prerequisite for the rest of Part 1 -- you can always come back to it later, once
you know you need direct AWS account control for Part 3.
{: .important }

<!-- prevnext:start -->

---

| [&larr; Previous: Databricks Platform Architecture]({{ '/01-databricks-fundamentals/02-introduction/databricks-platform-architecture/' | relative_url }}) | [Next: Creating Databricks Free Account &rarr;]({{ '/01-databricks-fundamentals/02-introduction/creating-databricks-free-account/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
