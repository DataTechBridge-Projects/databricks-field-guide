---
title: "Creating AWS Databricks Serverless Workspace"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-serverless-workspace/
read_minutes: 7
---

# Creating AWS Databricks Serverless Workspace
{: .no_toc }

*Estimated read: 7 min*

With your Databricks account created against your AWS Marketplace subscription, this lecture
creates your first **serverless workspace** -- the fastest path to a working environment, and
usually the right default unless you have a specific reason to need classic compute's networking
control (covered next lecture).

## Creating the workspace

1. In the Databricks account console, choose **Create workspace**.
2. Select **Serverless** as the workspace type.
3. Give it a name and pick a region -- for a learning setup, whichever region is closest to you or
   matches your AWS account's primary region is fine.
4. Confirm creation. Unlike classic compute, there's no VPC, subnet, or security group to
   configure -- Databricks provisions and manages the entire compute plane on your behalf, in its
   own account, exactly as described in the earlier architecture lecture.
5. Within a few minutes, the workspace is ready and you can sign in.

## Linking your AWS Marketplace subscription

If this is your first workspace on the account, you may be prompted to confirm the workspace is
linked to the Marketplace subscription from the previous lecture -- this is what routes usage
billing correctly. If you subscribed and set up the account in one continuous flow, this is often
already done automatically.

## Pay-as-you-go compute, in practice

Serverless workspaces bill by actual compute-second usage the moment you run something -- there's
no cluster to "leave running" in the classic sense, since Databricks manages scale-to-zero for you.
This is meaningfully different from classic compute, where an idle cluster can keep accruing cost
until it's terminated or an auto-termination policy kicks in.
{: .important }

## When serverless is (and isn't) the right choice

| Use serverless when | Use classic compute when |
|---|---|
| You want the fastest path to running code | You need specific instance types (e.g. GPU) |
| Your workload is ad hoc queries, dashboards, most day-to-day notebooks | You need custom VPC peering or on-prem connectivity |
| You don't need to control the underlying network | You're migrating workloads with existing networking dependencies |

For this guide's Part 1 and Part 2 content, serverless is sufficient throughout. See
[Databricks' workspace documentation](https://docs.databricks.com/aws/en/admin/workspace/) for the
full comparison and the account-console, API, and Terraform-based creation paths beyond the manual
console flow shown here.

Continue to
[Creating AWS Databricks Classic Workspace]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-classic-workspace/' | relative_url }})
to see the alternative -- even if you don't need it today, understanding when you would is worth
five more minutes.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating AWS Databricks Service Contract]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-service-contract/' | relative_url }}) | [Next: Creating AWS Databricks Classic Workspace &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-classic-workspace/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
