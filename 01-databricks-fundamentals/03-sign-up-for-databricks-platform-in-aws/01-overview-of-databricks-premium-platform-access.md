---
title: "Overview of Databricks Premium Platform Access"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 1
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/overview-of-databricks-premium-platform-access/
read_minutes: 4
---

# Overview of Databricks Premium Platform Access
{: .no_toc }

*Estimated read: 4 min*

This section walks through the path you'll need if you're preparing for Part 3's migration
content, or if you specifically want the classic-compute, your-own-AWS-account setup described in
the earlier architecture lecture. If you're staying on Free Edition for now, skip ahead to
[Working in Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }})
and come back here later.

## The shape of this section

Seven remaining lectures, in order:

1. Create an **AWS account**, if you don't already have one.
2. Create an **AWS IAM user** to administer the setup (rather than using the AWS root account for
   day-to-day work).
3. Set up **AWS cost monitoring** before you spend anything real -- a habit worth having before,
   not after, your first bill.
4. Subscribe to Databricks through the **AWS Marketplace**, creating your Databricks account
   contract.
5. Create a **serverless workspace** on that account.
6. Create a **classic workspace** on that account -- and understand when you'd actually reach for
   each.
7. **Delete and clean up** everything, so you're not paying for a workspace you built purely to
   learn from.

## Why go through this instead of just using Free Edition

Free Edition, from the earlier lecture, gets you a real workspace with zero AWS involvement. What
it deliberately doesn't give you is the thing a **paid, AWS-account-linked** workspace does:
direct control over the AWS account your data lives in -- your own S3 buckets, your own IAM roles,
your own VPC if you need one. That control is exactly what Part 3's migration content assumes,
because a real migration project always runs inside a customer's own AWS account, not a shared
free tier.

**Key term:** the AWS Marketplace path is a **billing mechanism**, not a different product --
Databricks itself is identical whether you subscribe directly or through the Marketplace. What
changes is that usage shows up on your AWS bill instead of a separate Databricks invoice. See
[Databricks pricing](https://www.databricks.com/product/pricing) for the underlying pay-as-you-go
and committed-use models this billing sits on top of.

## Cost-saving cleanup, up front

Before you start: every hands-on step in this section is designed to be **fully reversible**. The
last lecture, **Delete and Cleanup Databricks Workspace**, walks through tearing everything down.
If you're only doing this to learn the setup flow, budget five extra minutes at the end to run
that cleanup rather than leaving a workspace (and its underlying AWS resources) running.
{: .important }

<!-- prevnext:start -->

---

| [&larr; Previous: Sign up for Databricks Platform in AWS]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/' | relative_url }}) | [Next: Creating AWS Account &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-account/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
