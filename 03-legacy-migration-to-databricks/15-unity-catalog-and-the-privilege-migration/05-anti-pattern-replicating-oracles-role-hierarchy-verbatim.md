---
title: "Anti-Pattern: Replicating Oracle's Role Hierarchy Verbatim"
parent: "Unity Catalog and the Privilege Migration"
grand_parent: "Legacy Migration to Databricks"
nav_order: 5
permalink: /03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/anti-pattern-replicating-oracles-role-hierarchy-verbatim/
read_minutes: 3
---

# Anti-Pattern: Replicating Oracle's Role Hierarchy Verbatim
{: .no_toc }

*Estimated read: 3 min*

Faced with 500 Oracle roles and a deadline, the path of least resistance is tempting: script an export of `DBA_ROLES` and `DBA_ROLE_PRIVS`, generate one Unity Catalog group per Oracle role, and replay every `GRANT` verbatim against the equivalent catalog/schema path. It runs, it passes the functional cutover tests, and it is the single most common governance mistake in a lakehouse migration -- because it migrates the legacy system's *accumulated technical debt* along with its data, rather than migrating the *business intent* the roles were originally created to express.

Three concrete failure modes show up within months of a verbatim replication:

**Grant sprawl becomes group sprawl.** If `jsmith` had six roles in Oracle because access requests over the years were granted incrementally rather than through a clean role redesign, the migration now creates a Unity Catalog identity with six group memberships that nobody can explain either. The problem didn't get fixed; it got a new home.

**Nested role hierarchies don't map cleanly.** Oracle roles can grant other roles (`GRANT AR_CLERK TO AR_MANAGER`), building multi-level hierarchies that took years to accrete. Unity Catalog groups can nest too, but replicating a five-level-deep Oracle role tree verbatim just moves the "nobody can trace why this person has this access" problem into a system where the audit tooling (`information_schema`, system tables) is *better* at surfacing sprawl -- which means the sprawl gets found faster, usually during the first post-migration security review, and usually by someone other than the migration team.

**Dead roles get resurrected as dead groups.** Every decade-old Oracle instance has roles nobody has granted in years -- created for a project that shipped, a contractor who left, a report that was retired. A verbatim replication script doesn't know a role is dead; it just sees a role with members and privileges and faithfully recreates it, permanently importing historical cruft into a system that was supposed to be the clean slate.

| Approach | What it optimizes for | What it costs |
|---|---|---|
| Verbatim role → group replication | Migration speed, zero access-review conversations | Imports sprawl, nesting complexity, and dead grants permanently |
| Matrix-driven, tag-based redesign (previous two lectures) | A governance model sized to actual current access needs | Requires schema-owner review time before cutover |

{: .important }
> The tell that a migration defaulted to verbatim replication: the number of Unity Catalog groups after migration is within 10% of the number of Oracle roles before it. A real redesign should show meaningful compression -- if the privilege migration matrix and tagging exercise from the previous two lectures were actually applied, 500 roles collapsing to 40-60 groups plus a dozen tags is a realistic outcome, not 500 groups with new names.

The fix isn't more tooling, it's sequencing: the privilege migration matrix and tag taxonomy have to happen *before* the group-creation script runs, not after, because a mechanical export-and-replay will always be faster to write than a review -- and faster-to-write is exactly what makes it the default when a cutover date is looming. Treat "migrate every grant" and "migrate every grant *that a schema owner has confirmed is still needed*" as two different projects with two different timelines, and budget for the second one.

<!-- prevnext:start -->

---

| [&larr; Previous: 500 Roles to 12 Tags Without a Tagging Explosion]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/500-roles-to-12-tags-without-a-tagging-explosion/' | relative_url }}) | [Next: Check Your Knowledge &rarr;]({{ '/03-legacy-migration-to-databricks/15-unity-catalog-and-the-privilege-migration/check-your-knowledge/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->

