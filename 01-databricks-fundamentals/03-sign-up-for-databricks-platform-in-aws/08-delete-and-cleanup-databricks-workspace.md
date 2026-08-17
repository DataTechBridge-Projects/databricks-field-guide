---
title: "Delete and Cleanup Databricks Workspace"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 8
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/delete-and-cleanup-databricks-workspace/
read_minutes: 8
---

# Delete and Cleanup Databricks Workspace
{: .no_toc }

*Estimated read: 8 min*

If you built the serverless and classic workspaces purely to learn the setup flow, this lecture
tears them down -- and, more importantly, checks that the underlying AWS resources actually go
with them. Skip this only if you're genuinely keeping the workspace for ongoing use.

## Deleting a workspace

1. In the Databricks account console, select the workspace and choose **Delete**.
2. Confirm the deletion -- this is irreversible. Any notebooks, jobs, or Delta tables stored in
   the workspace's own storage are removed.
3. Repeat for each workspace you created (serverless and classic, if you made both).

## What deletion does *not* automatically remove

This is the part worth being deliberate about, because it's where a forgotten bill comes from:

- **A customer-managed VPC**, if you supplied one rather than using Quick start -- Databricks
  doesn't own it, so it doesn't delete it.
- **S3 buckets you created manually** for external locations (relevant once you reach the Unity
  Catalog section) -- workspace deletion removes the workspace's own root bucket reference, but
  buckets you created yourself outside that flow need manual cleanup.
- **CloudFormation stacks**, if Quick start used one to provision the cross-account IAM role and
  VPC -- check **AWS CloudFormation** in the console for a stack matching your workspace name and
  delete it explicitly if it's still there.
{: .important }

## Checking you're actually at zero

After deleting workspaces:

1. Go back to **AWS Billing -> Cost Explorer** (set up two lectures ago) and filter to the last
   24-48 hours, grouped by service.
2. Confirm no EC2, EBS, or S3 charges are still accruing that trace back to Databricks resources.
3. Check the **CloudFormation** console directly for any leftover stacks, and **EC2 -> Instances**
   for anything still running that you don't recognize.

## Canceling the AWS Marketplace subscription

If you're done with Databricks entirely (not just this session), the Marketplace subscription
itself -- separate from any individual workspace -- can be canceled from **AWS Marketplace ->
Manage subscriptions**. Canceling it stops the underlying account relationship; do this only once
you're certain you're finished, since it's a step above simply deleting a workspace.

## What you've now seen end to end

Across this section you've created an AWS account, an IAM admin user, a cost budget, a Databricks
Marketplace subscription, both workspace types, and now torn all of it back down -- the complete
lifecycle you'd walk a new team member through in a real onboarding. From here, whether you're
continuing on Free Edition or a fresh workspace of either type,
[Working in Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }})
is where hands-on work with notebooks, clusters, and code actually begins.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating AWS Databricks Classic Workspace]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-databricks-classic-workspace/' | relative_url }}) | [Next: Working in Databricks Workspace &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
