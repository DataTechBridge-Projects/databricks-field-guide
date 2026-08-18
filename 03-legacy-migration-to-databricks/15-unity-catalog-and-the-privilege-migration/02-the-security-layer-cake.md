---
title: "The Security Layer Cake"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 2
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-security-layer-cake/
read_minutes: 3
---

# The Security Layer Cake
{: .no_toc }

*Estimated read: 3 min*

An Oracle DBA debugging "why can't this user see this row" usually has one place to look: the grants and, if VPD (Virtual Private Database) policies are in play, the policy function attached to the table. In a lakehouse, the same question has four independent places to check, stacked like layers of a cake -- and a migration that only translates the top layer (table grants) and ignores the other three will pass every functional test and still fail the first real security review.

**Layer 1: Cloud IAM.** Before Unity Catalog even evaluates a grant, the compute resource itself needs permission to read the underlying cloud storage. This is the **storage credential** and **external location** layer from the previous lecture -- an IAM role scoped to an S3 bucket or prefix. Get this layer wrong (too broad) and Unity Catalog's grants become theater: a service principal with direct bucket access can bypass Unity Catalog entirely by reading the Parquet files straight off S3.

**Layer 2: Unity Catalog grants.** This is the layer legacy teams recognize immediately -- `GRANT SELECT ON catalog.schema.table TO group` -- the direct translation target for Oracle's `GRANT SELECT ON schema.table TO role`. It answers "can this identity touch this object at all," but nothing about which rows or columns within it.

**Layer 3: Row-level security.** A **row filter** is a SQL function attached to a table that Unity Catalog evaluates transparently on every query, silently dropping rows the caller shouldn't see -- the direct successor to Oracle VPD or a hand-rolled `WHERE region_id IN (SELECT ... FROM user_region_map)` predicate that used to get pasted into every report query. The difference that matters for a migration: in Oracle, that predicate lived in application code or a VPD policy function that could be forgotten on a new report; in Unity Catalog, the filter is bound to the table itself, so every query engine -- notebook, SQL warehouse, BI tool -- gets it automatically.

**Layer 4: Column masking.** A **column mask** is the same idea applied to a single column instead of a row: a function that transforms or nulls out a column's value based on who's asking, replacing the "create a masked view for the analyst role" pattern many Oracle shops built by hand for PII like SSNs or account numbers.

```sql
-- Row filter: restrict rows to the caller's region
CREATE FUNCTION region_filter(region_id STRING)
RETURN IF(IS_ACCOUNT_GROUP_MEMBER('region_admin'), true, region_id = current_user_region());

ALTER TABLE finance.gl.transactions SET ROW FILTER region_filter ON (region_id);

-- Column mask: show last 4 digits of an account number unless in the finance_admin group
CREATE FUNCTION mask_account(acct STRING)
RETURN IF(IS_ACCOUNT_GROUP_MEMBER('finance_admin'), acct, CONCAT('****', RIGHT(acct, 4)));

ALTER TABLE finance.gl.accounts ALTER COLUMN account_number SET MASK mask_account;
```

{: .important }
> Query results are the intersection of all four layers, not the union -- a user who passes the IAM and grant layers can still see zero rows if a row filter excludes them, and a grant migration that stops at layer 2 will look complete right up until someone notices the row filters and column masks never made the trip over from Oracle's VPD policies.

For a migration, this means the privilege migration matrix in the next lecture has to track more than table-level grants: any Oracle VPD policy or masked view needs its own line item, translated into a row filter or column mask rather than dropped. Full syntax and examples are in the [row and column filters documentation](https://docs.databricks.com/aws/en/data-governance/unity-catalog/row-and-column-filters); Section 16 goes deeper on ABAC patterns that generalize this beyond one-off functions.

<!-- prevnext:start -->

---

| [&larr; Previous: Unity Catalog Architecture]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/unity-catalog-architecture/' | relative_url }}) | [Next: The Privilege Migration Matrix &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-privilege-migration-matrix/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

