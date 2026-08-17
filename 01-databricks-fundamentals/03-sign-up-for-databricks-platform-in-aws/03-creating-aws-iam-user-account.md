---
title: "Creating AWS IAM User Account"
parent: "Sign up for Databricks Platform in AWS"
grand_parent: "Databricks Fundamentals"
nav_order: 3
permalink: /01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-iam-user-account/
read_minutes: 7
---

# Creating AWS IAM User Account
{: .no_toc }

*Estimated read: 7 min*

**IAM** (Identity and Access Management) is AWS's access-control system -- the direct analogue of
the user/role/GRANT system you managed in a legacy warehouse, except it governs an entire cloud
account, not just one database engine. This lecture creates the user you'll actually work as for
the rest of this section, instead of the root account from the previous lecture.

## Creating the user

1. In the AWS Console, go to **IAM -> Users -> Create user**.
2. Give it a clear name (e.g. `databricks-admin`) -- something that will still make sense to you
   (or a teammate) six months from now.
3. **Grant console access** if you'll sign in through the browser, with **"User must create a new
   password at next sign-in"** checked -- don't set a permanent password yourself.
4. **Attach permissions.** For this guide, attaching the AWS-managed `AdministratorAccess` policy
   is the pragmatic choice for a learning account. In a real organization, you'd instead put users
   in groups and attach least-privilege policies scoped to exactly what each group needs -- exactly
   the model Unity Catalog mirrors later for data access specifically.
5. **Enable MFA** for this user too, not just root.

```bash
# Equivalent via AWS CLI, if you prefer scripting the setup
aws iam create-user --user-name databricks-admin
aws iam attach-user-policy \
  --user-name databricks-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

## Why a dedicated user, not root

This is the same instinct behind never running production ETL jobs as a warehouse's `sa` or
`sys` account: the principle is **separation of the identity that can do anything from the
identity that does daily work**. If this IAM user's credentials are ever compromised, the blast
radius is contained to whatever policies you attached -- root's blast radius is the entire
account, unrecoverable in the worst case.

For the full official reference on IAM users, including permissions boundaries and the
console-vs-programmatic-access tradeoffs only briefly covered here, see AWS's own
[Create an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
documentation.

## What this user will do next

Everything from here forward in this section -- the Databricks Marketplace subscription, workspace
creation -- happens signed in as this IAM user, not root. Databricks itself will later create its
own IAM roles (covered in the platform architecture lecture's discussion of cross-account access)
to reach into your account for classic compute; this user is what sets that up, not what Databricks
uses at runtime.

<!-- prevnext:start -->

---

| [&larr; Previous: Creating AWS Account]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/creating-aws-account/' | relative_url }}) | [Next: Managing your AWS Cost &rarr;]({{ '/01-databricks-fundamentals/03-sign-up-for-databricks-platform-in-aws/managing-your-aws-cost/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
