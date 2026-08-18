---
title: "The Privilege Migration Matrix"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 3
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-privilege-migration-matrix/
read_minutes: 3
---

# The Privilege Migration Matrix
{: .no_toc }

*Estimated read: 3 min*

Every legacy grant that has accumulated over a decade -- `DBA_TAB_PRIVS`, `DBA_ROLE_PRIVS`, a spreadsheet someone maintains by hand -- needs to end up somewhere in Unity Catalog before cutover. The tool for that translation is a **privilege migration matrix**: one row per (legacy grantee, legacy object, legacy privilege), one target column for (Unity Catalog identity, Unity Catalog object, Unity Catalog privilege), and a status column tracking whether the row has been translated, reviewed, and applied.

Building the matrix starts with an inventory query against Oracle's data dictionary, not a guess:

```sql
-- Oracle: enumerate every object-level grant
SELECT grantee, owner, table_name, privilege
FROM dba_tab_privs
WHERE owner NOT IN ('SYS', 'SYSTEM')
ORDER BY grantee, owner, table_name;

-- Oracle: enumerate role membership, to expand roles into their granted privileges
SELECT grantee, granted_role
FROM dba_role_privs
ORDER BY grantee;
```

That export is almost always larger than anyone expects -- 500 per-table grants scattered across 40 roles is a typical shape for a decade-old finance schema -- because Oracle grants accrete: someone needed read access for one report in 2019, got a one-off `GRANT SELECT`, and it was never revoked. The matrix is where that accretion gets confronted rather than copied forward.

| Legacy Grantee | Legacy Object | Legacy Privilege | Target Identity | Target Object | Target Privilege | Status |
|---|---|---|---|---|---|---|
| AR_CLERK (role) | FIN.AR_INVOICES | SELECT, UPDATE | `finance_ar_clerks` (group) | `finance.accounts_receivable` (schema) | `USE SCHEMA`, `SELECT`, `MODIFY` | Reviewed |
| jsmith (user) | FIN.AR_INVOICES | SELECT | *(collapse into group above)* | -- | -- | Collapsed |
| RPT_VIEWER (role) | FIN.GL_SUMMARY_VW | SELECT | `finance_reporting` (group) | `finance.general_ledger` (schema) | `USE SCHEMA`, `SELECT` | Reviewed |

Two translation decisions do most of the work of shrinking that inventory. First, **collapse per-table grants to schema-level grants** wherever every table in a schema shares the same audience -- a `GRANT SELECT ON catalog.schema TO group` covers every current and future table in one statement, instead of 40 individual `GRANT SELECT ON table`. Second, **collapse individual-user grants into their group**, since Unity Catalog's best-practice guidance is to avoid granting directly to users at all; if `jsmith` only ever had the same access as the rest of the AR team, that row disappears into the group grant rather than becoming a durable per-user exception nobody remembers the reason for.

```sql
-- Unity Catalog: the schema-level grant that replaced 40 per-table GRANTs
GRANT USE CATALOG ON CATALOG finance TO `finance_ar_clerks`;
GRANT USE SCHEMA, SELECT, MODIFY ON SCHEMA finance.accounts_receivable TO `finance_ar_clerks`;
```

{: .important }
> Every row that gets collapsed or dropped needs a named reviewer and a reason recorded in the matrix -- "we think nobody uses this grant anymore" is not the same statement as "the schema owner confirmed this grant is dead," and the difference is what you'll be asked to produce in the first post-migration access audit.

Once the matrix is built, validate it two ways before applying it: run `SHOW GRANTS ON <object>` against the target catalog and diff the result against the matrix's target columns, and separately have the legacy schema owner sign off on every row marked "Collapsed" or "Dropped" -- the matrix is a design artifact, but the owner's approval is what makes it a migration decision rather than an assumption. See the [manage privileges guide](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/) for the full grant/revoke/show syntax.

<!-- prevnext:start -->

---

| [&larr; Previous: The Security Layer Cake]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/the-security-layer-cake/' | relative_url }}) | [Next: 500 Roles to 12 Tags Without a Tagging Explosion &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/500-roles-to-12-tags-without-a-tagging-explosion/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

