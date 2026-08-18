---
title: "500 Roles to 12 Tags Without a Tagging Explosion"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 4
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/500-roles-to-12-tags-without-a-tagging-explosion/
read_minutes: 3
---

# 500 Roles to 12 Tags Without a Tagging Explosion
{: .no_toc }

*Estimated read: 3 min*

The privilege migration matrix collapses per-table grants into schema-level group grants, but that alone still leaves you with as many Unity Catalog groups as the Oracle instance had roles -- if 500 Oracle roles map one-to-one to 500 Databricks groups, you haven't reduced the governance burden, you've just renamed it. The next compression step is **tags**: attribute-value pairs (`domain=finance`, `pii=true`, `tier=restricted`) attached to catalogs, schemas, tables, or columns, which let a single grant or a single row filter apply across every object carrying a tag rather than requiring one grant per object.

Where role explosion comes from in the first place: Oracle roles usually encode two things at once -- *what business function* someone performs (AR clerk, GL analyst) and *what sensitivity level* the data they touch carries (public, PII, restricted). Because those two dimensions were baked into a single role hierarchy, every new combination of function and sensitivity produced a new role: `AR_CLERK_PII`, `AR_CLERK_NO_PII`, `AR_MANAGER_PII`, and so on, until the role count outpaces the number of distinct people using them.

Tags let you separate those two dimensions instead of multiplying them together. A small number of **groups** (`finance_ar_clerks`, `finance_managers`) map to *who someone is*, and a small number of **tags** (`pii`, `domain`, `retention_tier`) map to *what the data is* -- and access policy is expressed as a rule that references both, rather than as one role per combination.

```sql
-- Tag a column as PII once
ALTER TABLE finance.accounts_receivable.customers
  ALTER COLUMN ssn SET TAGS ('pii' = 'true');

-- Tag an entire schema by business domain
ALTER SCHEMA finance.accounts_receivable SET TAGS ('domain' = 'finance');
```

With tags in place, a masking policy or a row filter (from the security layer cake two lectures back) can be written once against the tag rather than against every individual table:

```sql
-- One masking function, applied to every column tagged pii=true across the metastore,
-- instead of one ALTER COLUMN ... SET MASK per table
CREATE FUNCTION mask_pii(val STRING)
RETURN IF(IS_ACCOUNT_GROUP_MEMBER('pii_reviewers'), val, '***REDACTED***');
```

That's how "500 roles" becomes "12 tags": the 500 roles mostly encoded a handful of real sensitivity tiers and a handful of real business domains, repeated in every combination. Twelve tags -- three or four sensitivity levels crossed with three or four domains -- express the same policy surface without a combinatorial explosion of groups.

{: .important }
> A tagging model can explode just as badly as a role hierarchy if tags are created ad hoc per project instead of from a governed taxonomy -- agree on the fixed set of tag keys and allowed values (via Unity Catalog's governed tags) *before* teams start tagging, or you'll trade 500 roles for 500 free-text tag values.

The payoff shows up at audit time: "show me every table containing PII" is a single query against `information_schema.column_tags` instead of a manual crawl through hundreds of grants. See the [tags documentation](https://docs.databricks.com/aws/en/data-governance/unity-catalog/tags) for governed tags and tag-based access policies, and the [best practices guide](https://docs.databricks.com/aws/en/data-governance/unity-catalog/best-practices) for sizing a group and tag taxonomy before rollout.

<!-- prevnext:start -->

---

| [&larr; Previous: The Privilege Migration Matrix]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-privilege-migration-matrix/' | relative_url }}) | [Next: Anti-Pattern: Replicating Oracle's Role Hierarchy Verbatim &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/anti-pattern-replicating-oracles-role-hierarchy-verbatim/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

