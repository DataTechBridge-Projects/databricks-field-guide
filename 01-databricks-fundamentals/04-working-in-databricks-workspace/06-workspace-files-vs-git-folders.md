---
title: "Workspace Files vs Git Folders"
parent: "Working in Databricks Workspace"
grand_parent: "Databricks Fundamentals"
nav_order: 6
permalink: /01-databricks-fundamentals/04-working-in-databricks-workspace/workspace-files-vs-git-folders/
read_minutes: 13
---

# Workspace Files vs Git Folders
{: .no_toc }

*Estimated read: 13 min*

Everything you've written so far this section lives as plain **workspace files**, with
Databricks' own automatic revision history behind them. That's fine for exploration. It's not
where production pipeline code should live long-term -- this lecture covers **Git folders**, the
version-controlled alternative, and when to reach for which.

## Workspace files: the default

A notebook or file created directly in the workspace sidebar is a **workspace file** -- stored in
Databricks' own backing storage, with automatic, Databricks-managed revision history (a list of
past versions you can browse and restore, distinct from Git). No repository, no branches, no
commit messages -- just "what did this notebook look like an hour ago."

**Good for:** interactive exploration, one-off analysis, anything you're not planning to promote
into a scheduled, production pipeline.

**Not good for:** anything you want code review on, anything that needs to move through
dev -> uat -> prod as a coherent, versioned unit, or anything more than one person needs to
collaborate on without risking silently overwriting each other's changes.

## Git folders: version control, properly

**Git folders** (formerly called "Repos") integrate an actual Git repository -- GitHub, GitLab,
Azure DevOps, Bitbucket -- directly into the workspace file tree. A Git folder behaves like a
regular checkout: pull, branch, commit, push, and open pull requests, all from within the
Databricks UI (or, for CI/CD, via the Databricks CLI and REST API, covered later in Part 2).

```text
Workspace
├── Repos (Git folders)
│   └── steprightproject/          <- an actual git clone, branch-aware
│       ├── pipelines/
│       │   └── bronze_orders.py
│       └── common/
│           └── setup.py
└── (regular workspace files, no version control)
```

## Setting up a Git folder

1. In the workspace sidebar, choose **Git folders -> Add repo**.
2. Provide the repository URL and authenticate (a **personal access token** or, for GitHub, the
   Databricks GitHub App integration).
3. Choose a branch to check out. The folder now mirrors that branch, with a **branch selector** in
   the notebook UI itself -- switching branches from inside Databricks, not a separate terminal.
4. Commit and push changes directly from the notebook's Git panel, same as any Git client.

## The comparison that matters for this guide

| | Workspace files | Git folders |
|---|---|---|
| Version history | Databricks-managed, no branches | Full Git history, branches, PRs |
| Collaboration model | Real-time co-editing, one shared copy | Branch-per-feature, merge via PR |
| CI/CD integration | None | Native -- the same repo your CI pipeline builds from |
| Promotion across environments | Manual copy/export | Deploy the same commit/tag to dev, uat, prod |
| Best for | Exploration, ad hoc analysis | Anything production-bound |

**Key term:** the pattern of developing in a branch, opening a pull request, and deploying the
merged result to each environment in turn is exactly the promotion model
[Part 2's StepRight project]({{ '/02-stepright-capstone-project/' | relative_url }}) builds hands-on, including a real
CI/CD pipeline in its final section -- Git folders are the notebook-side half of that story.

## A practical rule of thumb

Prototype in a plain workspace notebook if you're still figuring out an approach. The moment
you're confident enough in a transformation to want it scheduled, reviewed, or shared with a
teammate as something they can build on, move it into a Git folder -- don't wait until "it's
finished," because by then the exploratory version has usually accumulated exactly the kind of
undocumented, unreviewed logic a legacy warehouse team learns to distrust in ad hoc scripts.
{: .important }

For the full official reference on Git folders, including CI/CD automation and advanced Git
configuration beyond this lecture's scope, see
[Databricks Git folders documentation](https://docs.databricks.com/aws/en/repos/).

<!-- prevnext:start -->

---

| [&larr; Previous: How to Debug Notebooks]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/how-to-debug-notebooks/' | relative_url }}) | [Next: Databricks Compute Cluster &rarr;]({{ '/01-databricks-fundamentals/04-working-in-databricks-workspace/databricks-compute-cluster/' | relative_url }}) |
|:---|---:|

<!-- prevnext:end -->
