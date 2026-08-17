---
title: "Creating AWS Databricks Service Contract"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 5
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-service-contract/
read_minutes: 11
---

# Creating AWS Databricks Service Contract
{: .no_toc }

*Estimated read: 11 min*

With an AWS account, an IAM admin user, and a budget alert in place, this lecture subscribes to
Databricks through the **AWS Marketplace** -- the step that actually creates your billable
Databricks account.

## Subscribing through AWS Marketplace

1. Sign in to the AWS Console as your IAM admin user and go to **AWS Marketplace ->
   Discover products**.
2. Search for **Databricks** and open the listing.
3. Choose **Subscribe** or **Continue to Subscribe**, and accept the contract terms. This step
   doesn't provision anything yet -- it authorizes your AWS account to be billed for Databricks
   usage, the same way subscribing to any other Marketplace software product would.
4. Once subscribed, choose **Set up your account** (or the equivalent "Continue to configuration"
   action) to be redirected into Databricks' own account setup flow.
5. Complete the Databricks-side account setup: account name, primary contact, and (if prompted)
   which AWS region your first workspace should live in.

**Key term:** this creates a **Databricks account** -- the top-level billing and identity entity
from the earlier architecture lecture -- distinct from any individual **workspace**, which you'll
create in the next two lectures. One account can hold many workspaces.

## What "AWS Marketplace billing" actually changes

Functionally, nothing about Databricks itself differs based on how you subscribed. What changes is
purely commercial:

| | Direct Databricks contract | AWS Marketplace subscription |
|---|---|---|
| Invoicing | Separate Databricks invoice | Consolidated into your AWS bill |
| Procurement | New vendor relationship | Uses existing AWS spend commitments, if any |
| Setup | Databricks-managed signup | AWS Marketplace subscription flow |

For an enterprise already deep into AWS spend commitments, routing Databricks usage through the
Marketplace can count toward existing commitments -- worth knowing if you're the person who'll
eventually justify this contract to a finance team, which is exactly the audience for
[Part 3's TCO content]({{ '/03-legacy-migration-to-databricks/03-the-3-r-decision-and-the-tco-that-convinces-the-cfo/' | relative_url }}).

## The 14-day credit window

New AWS Marketplace subscriptions to Databricks typically come with a limited trial credit window
(commonly 14 days) before pay-as-you-go billing fully applies. Don't build your learning schedule
around finishing everything inside that window -- pay-as-you-go rates after it are still metered by
actual usage, not a flat fee, so the real cost driver is whether you tear resources down when
you're done (the final lecture in this section), not which side of the trial window you're on.
{: .important }

## If the Marketplace listing isn't visible

Marketplace product visibility can occasionally depend on your AWS account's region or
organizational Marketplace settings. If you can't find the Databricks listing, confirm you're
signed in with an account that has Marketplace subscription permissions (part of the
`AdministratorAccess` policy from the IAM lecture), and that you're searching from the AWS
Marketplace console rather than a general AWS service search.

Once your contract is active, move on to
[Creating AWS Databricks Serverless Workspace]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-serverless-workspace/' | relative_url }}).

<!-- prevnext:start -->

---

| [&larr; Previous: Managing your AWS Cost]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/managing-your-aws-cost/' | relative_url }}) | [Next: Creating AWS Databricks Serverless Workspace &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-serverless-workspace/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
