---
title: "Creating AWS Account"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 2
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-account/
read_minutes: 7
---

# Creating AWS Account
{: .no_toc }

*Estimated read: 7 min*

If you already run production workloads on AWS, you can skip straight to the next lecture on IAM
users -- use an existing account, not a brand-new one, so this doesn't collide with your
organization's account structure. This lecture is for a genuinely new AWS account, created purely
for working through this guide.

## Signing up

1. Go to the AWS sign-up flow and provide an email address and account name. This becomes your
   **root user** -- the single most privileged identity on the account, and one you should almost
   never use for day-to-day work after this setup is done.
2. **Enter payment information.** AWS requires a card even for free-tier usage, to verify identity
   and cover any usage beyond free-tier limits.
3. **Verify your identity** via phone (SMS or voice call, your choice).
4. **Choose a support plan.** The free **Basic** plan is enough for everything in this guide --
   don't be upsold into a paid plan you don't need yet.

## The account you get

A brand-new AWS account starts empty: no IAM users beyond root, no resources, no existing billing
history. That's deliberate -- it means you're not inheriting anyone else's configuration or
security posture. If this is a personal account you're using purely to work through this guide,
this is the right starting point.

## Root user hygiene, immediately

Before you do anything else in this new account:

- **Enable MFA (multi-factor authentication) on the root user.** The root user can do literally
  anything in the account, including deleting it -- protecting it is the single highest-leverage
  security step available to you.
- **Do not create access keys for the root user.** You won't need programmatic root access for
  anything in this guide, and long-lived root credentials are one of the most common causes of
  serious AWS account compromise.
- Plan to create a dedicated **IAM user** for actual administration -- covered in the next lecture
  -- and stop using root entirely once that user exists.
{: .important }

## Coming from a legacy on-prem or single-warehouse world

If your prior experience is entirely on-prem or inside a single managed warehouse account, the
concept worth internalizing here is that **an AWS account is a hard security and billing
boundary**, roughly analogous to a completely separate database instance with its own
administrator, not a schema or a role inside a shared instance. Everything you do in the rest of
this section -- IAM users, cost budgets, the Databricks Marketplace subscription -- lives inside
this one boundary.

Once your account exists and root is secured, move on to
[Creating AWS IAM User Account]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-iam-user-account/' | relative_url }}).

<!-- prevnext:start -->

---

| [&larr; Previous: Overview of Databricks Premium Platform Access]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/overview-of-databricks-premium-platform-access/' | relative_url }}) | [Next: Creating AWS IAM User Account &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-iam-user-account/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
