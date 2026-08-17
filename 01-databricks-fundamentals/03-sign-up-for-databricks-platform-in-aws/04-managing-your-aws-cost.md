---
title: "Managing your AWS Cost"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 4
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/managing-your-aws-cost/
read_minutes: 8
---

# Managing your AWS Cost
{: .no_toc }

*Estimated read: 8 min*

Before subscribing to anything that spends money, set up cost visibility. In a legacy warehouse
you might never have thought about this -- the hardware was already bought, or the license was a
fixed annual number. Cloud billing is metered and continuous, and a forgotten running cluster is a
genuinely common way to get a surprising invoice.

## The Billing and Cost Management console

AWS's **Billing and Cost Management** console is where every dollar spent on the account is
visible, broken down by service, by day, and (once tagged) by project or team. Two things worth
setting up immediately:

1. **Cost Explorer** -- a visual breakdown of spend over time, filterable by service. Useful for
   spotting "wait, why did EC2 spend jump yesterday" before it becomes a monthly surprise.
2. **AWS Budgets** -- proactive alerting, covered next.

## Setting a budget with email alerts

1. In the Billing console, go to **Budgets -> Create budget**.
2. Choose a **cost budget** with a fixed monthly amount -- something comfortably above what you
   expect to spend learning this guide's hands-on sections, so an alert means "something is
   actually wrong," not "you used the platform normally."
3. Set an **alert threshold**, e.g. 80% of budget, actual or forecasted spend.
4. Add your **email address** as the notification target.

```text
Example: Monthly cost budget = $20
Alert at: 80% actual spend ($16) -> email notification
Alert at: 100% forecasted spend -> email notification
```

AWS Budgets data updates up to three times a day, with a delay of roughly 8-12 hours between usage
and the budget reflecting it -- it's a safety net for catching sustained overspend, not a real-time
meter. See AWS's own
[Managing your costs with AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
documentation for the full set of budget types, including usage budgets and Reserved
Instance/Savings Plan coverage tracking, beyond the simple cost budget this guide needs.

## Net cost against credits and free tier

If you're on a new account, AWS's free tier and any promotional credits apply automatically --
your **net cost** (what actually gets charged) can be $0 well past your budget threshold if
you're within those limits. Don't let a budget alert alone convince you something's broken; check
the Cost Explorer breakdown to see whether you're still inside free-tier usage before assuming
you've been billed.
{: .important }

## Why this matters specifically for the next lectures

The lectures immediately following this one -- creating a Databricks service contract and standing
up serverless and classic workspaces -- are the first things in this guide that can generate real
AWS spend (compute for clusters, storage for workspace data). Having a budget and alert already in
place means you'll know immediately if the cleanup lecture at the end of this section didn't fully
tear something down.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating AWS IAM User Account]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-iam-user-account/' | relative_url }}) | [Next: Creating AWS Databricks Service Contract &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-service-contract/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
