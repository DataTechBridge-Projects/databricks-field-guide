---
title: "Creating Databricks Free Account"
parent: "Introduction"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/02-introduction/creating-databricks-free-account/
read_minutes: 4
---

# Creating Databricks Free Account
{: .no_toc }

*Estimated read: 4 min*

If you're following the Free Edition path from the previous lecture, this walks through account
creation end to end. It takes a few minutes and needs nothing but an email address.

## Sign-up steps

1. Go to [Databricks Free Edition](https://www.databricks.com/learn/free-edition) and choose
   **Sign up for Free Edition**.
2. **Verify your email.** Databricks sends a confirmation link -- click it before continuing.
3. **Set your account name and country.** This is used for your workspace URL and regional data
   handling; it's not something you'll change often, so pick deliberately rather than accepting a
   placeholder.
4. **Complete the verification puzzle.** A standard bot-prevention step -- nothing Databricks-
   specific, just don't be surprised by it.
5. You land in a **fully provisioned, serverless workspace** -- no cluster to configure, no region
   to pick manually. This is the serverless compute model from the earlier architecture lecture in
   practice: the compute plane is entirely Databricks-managed, so there's no VPC or instance type
   for you to think about at all.

## First things to customize

Once you're in the workspace:

- **Workspace theme.** Light or dark, under your user settings -- purely cosmetic, but worth
  setting before you spend hours reading notebook output.
- **Editor theme and font size.** The notebook code editor has its own separate appearance
  settings, distinct from the workspace theme -- easy to miss on first login.
- **Your first notebook.** Create one immediately, even empty, so you have a working cell to
  experiment in as you read the rest of this part. Attach it to the default serverless compute --
  there's no cluster picker to worry about on Free Edition.

## What's different from the paid AWS path

Everything you just did happened without touching AWS at all -- no account, no IAM user, no
billing console. That's the entire point of Free Edition: it trades the account-level control
you'll want for Part 3's migration work for a genuinely zero-friction start. When you eventually
do walk through Section 3's AWS-based setup, notice how many more decisions it asks you to make
(region, IAM permissions, workspace storage bucket) -- those are the same decisions Free Edition
made for you invisibly.

You now have a working workspace. The next section,
[Sign up for Databricks Platform in AWS]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/' | relative_url }}),
is optional if you're staying on Free Edition for now -- skip ahead to
[Working in Databricks Workspace]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/' | relative_url }})
if you'd rather start writing code immediately.

<!-- prevnext:start -->

---

| [&larr; Previous: Databricks Platform Access - Paid vs Free]({{ '/01-databricks-fundamentals/02-introduction/databricks-platform-access-paid-vs-free/' | relative_url }}) | [Next: Sign up for Databricks Platform in AWS &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
