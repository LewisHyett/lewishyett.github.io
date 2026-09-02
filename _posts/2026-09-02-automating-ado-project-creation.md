---
layout: post
title: "Automating Azure DevOps Project Creation"
date: 2026-09-02 09:00:00 +0000
categories: [devops]
tags: [azure-devops, automation, powershell, pipelines]
---

## TL;DR

Every new customer needed the same Azure DevOps project setup: a specific
process template, a fixed area path tree, organisation group rules, repository
and query permissions, and a full set of blocking branch policies. Doing it by
hand took about 15 minutes of clicking through settings pages and was easy to
get subtly wrong. I moved the whole runbook into a PowerShell script driven by the
Azure DevOps REST API and wrapped it in a manual-trigger pipeline. Now creating
a project is: open the pipeline, type a name, click **Run**, wait about a
minute. Every project comes out identical.

## The problem: a runbook I kept re-running by hand

We stand up a new Azure DevOps project whenever we take on a customer. The
project itself is one click, but a *usable* project is not. The standard setup
needs:

- A specific **process template** with Git version control
- A couple of organisation-level **group rules** updated so their members get
  Project Contributors access on the new project automatically
- A fixed **area path hierarchy** — two top-level streams, each with the same
  three child nodes
- A delivery security group granted edit/create/delete on the area root, plus
  create/delete/rename/unlock on all repositories
- The default team reconfigured: an extra backlog level enabled, default area
  set to include sub-areas
- **Shared Queries** sub-folders for bug and issue tracking, and the delivery
  group given contribute/delete/manage-permissions on the folder
- A full set of **blocking branch policies** on `main` and `release/*` — minimum
  reviewers, linked work items, comment resolution, auto-included reviewers, and
  a locked-down merge strategy (rebase-and-fast-forward on `main`, squash on
  `release/*`)

Every one of those is a different settings page. Done manually it is roughly 15
minutes of careful clicking, and the failure mode is quiet: you forget to tick
"include sub-areas", or you set the merge strategy on `main` but not on
`release/*`, and nobody notices until months later when a branch policy behaves
unexpectedly.

After doing this by hand one too many times, I decided to encode the runbook
once and never click through it again.

## The approach: one script, one pipeline

> A sanitised, runnable copy of both files — with every organisation-, customer-,
> and team-specific value replaced by a described placeholder — is on GitHub:
> [LewisHyett/BCExamples / ADO-Project-Creation](https://github.com/LewisHyett/BCExamples/tree/main/ADO-Project-Creation).

The solution has two files:

```
repo/
├── azure-pipelines.yml   # manual-trigger pipeline definition
└── scripts/
    └── New-Project.ps1   # the automation, top to bottom
```

### The script

`New-Project.ps1` is a single PowerShell script that walks the runbook in order,
calling the Azure DevOps REST API for each step. It takes an org URL, a PAT, a
project name, and an optional description. Everything else — process name, group
paths, default branch — is a parameter with a sensible default, so the common
case needs no extra input.

It runs in two modes and detects which one it is in:

- **Locally** — prints the plan and asks for confirmation before touching
  anything. Good for testing and one-offs.
- **Inside a pipeline** — detected via the `TF_BUILD` environment variable. No
  prompt, and log output switches to Azure DevOps logging commands
  (`##[section]`, `##vso[task.logissue]`) so the run page gets collapsible
  sections and surfaced warnings.

A small `Invoke-Ado` helper wraps `Invoke-RestMethod` with the auth header,
JSON serialisation, and error unwrapping so that an API failure produces a
readable message instead of a raw HTTP exception.

### The pipeline

`azure-pipelines.yml` is deliberately minimal:

```yaml
trigger: none    # never runs on commits
pr: none

parameters:
  - name: ProjectName
    displayName: "Project name"
    type: string
  - name: ProjectDescription
    displayName: "Description (optional)"
    type: string
    default: " "

pool:
  vmImage: windows-latest

variables:
  - group: <variable group holding the PAT as a secret>
  - name: OrgUrl
    value: "https://dev.azure.com/<your organisation name>"
```

The single job checks out the repo and runs the script with `pwsh: false`
(Windows PowerShell 5.1, matching where it was tested). The PAT lives in a
variable group and is passed to the script through an **environment variable**,
not a command-line argument, so it never lands in the build logs.

## What each step actually does

The script follows the runbook step for step:

| # | Step | API surface |
|---|------|-------------|
| 1 | Create the project, then poll the operation until provisioning succeeds | `_apis/projects`, `_apis/operations` |
| 2 | Patch the group rules to add a `ProjectContributor` entitlement for the new project | `vsaex.dev.azure.com/_apis/GroupEntitlements` |
| 3 | Create the area path tree and grant the delivery group edit/create/delete on the root | `_apis/wit/classificationnodes/areas`, CSS security namespace |
| 4 | Enable the extra backlog level and set the team default area to include sub-areas | `_apis/work/teamsettings` |
| 5 | Grant the delivery group create/delete/rename/unlock across all repos | Git security namespace |
| 6 | Apply blocking branch policies to `main` (rebase-FF merge) | `_apis/policy/configurations` |
| 7 | Apply the same policy set to `release/*` (squash merge), matched by prefix | `_apis/policy/configurations` |
| 8 | Create the Shared Queries sub-folders and grant folder permissions | `_apis/wit/queries/{path}`, WIQ security namespace |

### The parts that took the longest to get right

**Permissions are bitmasks against security namespaces.** Azure DevOps does not
have a friendly "grant this group these permissions" API. You POST an access
control entry to a namespace GUID, with a `token` identifying the resource and
an integer `allow` mask. So area paths become:

```powershell
# CSS namespace (classification nodes): Edit=2, CreateChildren=4, Delete=8  =>  14
$cssNamespaceId = "<GUID of the CSS security namespace>"   # from _apis/securitynamespaces
$rootToken      = "vstfs:///Classification/Node/$($rootArea.identifier)"
$areaAllowMask  = 2 + 4 + 8
```

Each namespace GUID is a fixed Azure DevOps constant you look up once from
`_apis/securitynamespaces` (or the docs) and then hard-code. Repositories use a
different namespace and different bits
(`Create=256, Delete=512, Rename=1024, RemoveOthersLocks=4096` → `5888`), and
Shared Queries use a third, with a token shaped like `$/{projectId}/{folderId}`
where the folder GUID has to be fetched first. Each one was a small research
project.

**Group rules need a raw JSON Patch string.** PowerShell's `ConvertTo-Json`
unwraps single-element arrays, which breaks the JSON Patch document the
`GroupEntitlements` endpoint expects. The fix was to build that one body as a
here-string literal rather than convert an object.

**Project-level branch policies use a null repository ID.** Setting
`scope.repositoryId = $null` makes a policy apply to every repo in the project,
including ones created later — which is what you want for a template.

**Idempotency.** The script checks for an existing project and continues with
configuration if it finds one. Area and query-folder creation catch the
"already exists" conflict and fetch the existing node instead of failing. So a
re-run repairs a half-configured project rather than blowing up.

## The payoff

When the pipeline finishes it uploads a markdown summary to the run page listing
everything it configured, and publishes the new project URL as an output
variable so a later job could chain off it.

The practical difference:

- **Time:** ~15 minutes of manual clicking → ~1 minute, most of it unattended
- **Consistency:** every project is byte-for-byte the same setup
- **Onboarding:** "run this pipeline" instead of a page of runbook screenshots
- **Auditability:** the runbook is now code in a repo with history, not tribal
  knowledge

The PAT still needs a fairly broad scope set (Project & Team, Code, Graph,
Identity, Member Entitlement Management, Security — all manage/write level),
which is the main thing to keep an eye on. It lives in a variable group with
restricted access and the pipeline is manual-trigger only, with a permissions
group controlling who can run it.

## What I'd do next

- Move from a PAT to a service connection with a managed identity
- Add a dry-run parameter that logs intended API calls without making them
- Emit the summary as a proper work item or wiki page in the new project so the
  configuration record lives with the project itself

But even as it stands, the tedious thing is gone. That was the whole point.

---

**Sample code:** [LewisHyett/BCExamples / ADO-Project-Creation](https://github.com/LewisHyett/BCExamples/tree/main/ADO-Project-Creation)
— the pipeline definition and the full PowerShell script, sanitised, with a
README explaining every placeholder and the required PAT scopes.
