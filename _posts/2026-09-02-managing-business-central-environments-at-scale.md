---
layout: post
title: "One Console for Every Customer Business Central Environment"
date: 2026-09-02 09:00:00 +0000
description: >-
  A walkthrough of the Environment Manager: a single Business Central console for
  viewing every customer tenant, running bulk operations, scheduling upgrades,
  deploying extensions, and keeping a full audit trail.
categories: [business-central]
tags: [business-central, managed-services, multi-tenant, alm, admin-centre]
---

## TL;DR

If you look after Business Central for more than a handful of customers, every
tenant is its own login and every upgrade, app update, or sandbox copy is the
same short click sequence in a different browser tab. After enough evenings of
that, I started building a side project to fix it for myself: the **Environment
Manager**, a single console *inside* Business Central. One list of every customer
tenant, environment, and installed app; bulk operations across many
environments in one pass; scheduled upgrades and extension rollouts; and a single
action log that records what ran, against what, by whom, and when. This post is a
tour of what I built and why it is put together the way it is.

![Customer Tenants](/assets/images/EnvironmentManager/customer-tenants.png){: width="1742" height="182"}

## The problem with one-tenant-at-a-time

Managing environments by hand does not fail dramatically. It just quietly stops
scaling, and I kept running into the same four things.

- **Every tenant is an island.** There is no single view of which customers are
  near their storage quota, which environments are a version behind, or which app
  updates are outstanding across the estate.
- **Routine work multiplies.** Rolling one app update to twelve customers is the
  same dialog, twelve times, with twelve chances to pick the wrong environment.
- **Slow operations pull you back.** A copy or a restore can take thirty minutes
  or more, and each one is a portal tab you keep flicking back to instead of
  getting on with something else.
- **Activity is scattered.** The admin centre records what happened to each
  environment, but per environment, per tenant. There is no single place to see
  everything that has run across the whole estate this week, or what is running
  right now.

## The idea: a control panel that lives in Business Central

I wanted the tool to live where I already spend my day rather than in yet another
portal, so I built it as a set of pages inside Business Central, backed by two
things working together:

1. **A local cache** — copies of every customer tenant, environment, installed
   app, and storage figure, kept current by background jobs that run on a
   schedule.
2. **The live Microsoft APIs** — the Partner Centre API and the Business Central
   Admin Centre API remain the source of truth. The console never replaces them;
   it orchestrates them and records what happened.

That split ended up being the whole design in one sentence: **read from the cache
so the UI is fast, act through the APIs so the changes are real, and log
everything in between.**

```mermaid
flowchart TB
    subgraph BC["Business Central — the Environment Manager"]
        UI["Console pages<br/>Tenants · Environments · Apps · Schedule<br/>Deployments · Action Log · Partner Setup"]
        OPS["Operation layer<br/>validate → build request → execute → poll"]
        CACHE[("Local cache<br/>tenants · environments · apps · storage<br/>action log · deployments")]
        JOBS["Scheduled jobs<br/>Sync Customers (daily) · Refresh Environments (2h)<br/>Get Storage (6h) · Validate Access · Action Log poller<br/>Deploy Extensions dispatcher"]
    end

    subgraph MS["Microsoft cloud"]
        PC["Partner Centre API"]
        AC["BC Admin Centre API"]
        FEED["Package feed<br/>(.app + dependencies)"]
        ENV["Customer environments"]
    end

    UI -->|read| CACHE
    UI -->|start operation| OPS
    OPS --> CACHE
    OPS -->|act| AC
    JOBS --> OPS
    JOBS -->|discover customers| PC
    JOBS -->|refresh data| AC
    JOBS -->|pull packages| FEED
    JOBS --> CACHE
    PC -.->|customer list| CACHE
    AC -.->|environments · apps · storage| CACHE
    AC --> ENV
```

## Where the customer list comes from: syncing from Partner Centre

The console is only useful if it knows about every customer, so the first job it
runs is discovery. A scheduled **Sync Customers** job calls the Partner Centre
API once a day and writes the result into the local Customer Tenant table —
adding tenants that are new, updating ones that have changed. I never type a
customer in by hand; if they are in Partner Centre, they show up in the console
the next morning.

Partner Centre access uses Microsoft's Secure Application Model, so there is a
one-time consent step that produces a refresh token. After that the token rotates
itself on every use and I do not touch it again unless Partner Centre calls start
failing.

Discovery only gets you the *list*, though. Reading a tenant's environments and
storage needs separate per-tenant credentials against the Admin Centre API, and
that is the one part still done by hand: for each newly discovered tenant I set
its authentication — either its own app registration or a shared one reused
across several customers. A background **Validate Tenant Access** job then checks
each tenant and marks it 🟢 Accessible, 🟡 Access Denied, or ⚪ Unknown.

That status matters because the environment and storage refresh jobs **only
process tenants marked Accessible**. A tenant stuck on Access Denied or Unknown
is a visible flag on the tenant list telling me its credentials need attention —
nothing about that customer refreshes until it is fixed.

![API Access Status](/assets/images/EnvironmentManager/APIAccessStatus.png){: loading="lazy" decoding="async" width="294" height="242"}

## What you can see

Before any of the operations, the console just gives me a single place to look at
the estate. That part alone earned its keep.

- **Customer tenants.** Every customer with capacity and storage summaries,
  colour-coded so anything near a quota stands out.
- **Environments.** Every environment across every tenant, with its type, current
  version, and whether an update is waiting. Production environments are shown in
  bold; an upgrade due within a few days shows amber.
- **Apps per environment.** What is installed, and what has a newer version
  available.

![Customer Environments](/assets/images/EnvironmentManager/CustomerEnvironments.png){: loading="lazy" decoding="async" width="1788" height="347"}

![Capacity and storage columns](/assets/images/EnvironmentManager/CapacityFields.png){: loading="lazy" decoding="async" width="599" height="297"}

![Environment Apps](/assets/images/EnvironmentManager/EnvironmentApps.png){: loading="lazy" decoding="async" width="1799" height="329"}

The refresh happens on its own. Scheduled background jobs refresh environment
details every couple of hours and check storage a few times a day, on top of the
daily customer sync. New environments simply appear. You can also refresh any
single tenant or environment on demand when you need the very latest.

## What you can do: operations

Each operation follows the same pattern: select one or more rows, set only the
options that matter, choose whether to run now or leave it **Pending**, then
monitor it in the Action Log. The available environment operations are:

![Environment Operations](/assets/images/EnvironmentManager/EnvironmentOperationsRibbon.png){: loading="lazy" decoding="async" width="805" height="107"}

![Copy Environment](/assets/images/EnvironmentManager/CopyEnvironment.png){: loading="lazy" decoding="async" width="646" height="261"}

| Operation | What it does |
|---|---|
| Copy | Creates a new environment from an existing one, with a chosen name and environment type. |
| Restore | Creates an environment from a selected point in another environment's restore history. |
| Recover | Brings a soft-deleted environment back before permanent deletion. |
| Delete | Removes an eligible environment, with extra protection around production. |
| Set target version and date | Chooses the next platform version and when the upgrade should run. |
| Set update time | Defines the preferred daily window for updates. |
| Reschedule upgrade | Moves an upgrade that already has a date, with an option to ignore the normal window. |
| Set app update cadence | Controls whether apps update by default or alongside major or minor environment upgrades. |
| Deploy extensions | Uploads an `.app` file or deploys selected products from the package feed. |

Apps have their own focused actions once you drill into an environment:

| Operation | What it does |
|---|---|
| Refresh apps | Reloads the installed apps and available versions for the selected environments. |
| Upgrade apps | Updates selected apps that have a newer Microsoft-managed version available, now or on a schedule. |
| Cancel app update | Cancels a scheduled app update before it runs. |
| Uninstall apps | Removes selected apps, optionally with dependants and app data. |
| Upgrade from package feed | Rolls selected per-tenant extensions forward to the latest approved package and resolves dependencies. |

Because the actions start from selected rows, the same choices can be applied to
many environments or apps in one pass rather than repeated customer by customer.

## The spine: one action log for everything

Every operation — a copy, a restore, an app update, an extension deployment — is
written to a single **Action Log** as it happens. Newest first, with the status,
the operation type, a plain-language description, what it ran against, who ran it,
and when.

- 🟡 **In Progress** — the API is working on it
- 🟢 **Completed** — done
- 🔴 **Failed** — with the actual error message, plus the request that was sent
  and the response that came back
- ⏸️ **Pending** — logged but not sent yet
- 🕓 **Queued** — waiting for another operation on the same environment to finish
  first

The log is pruned automatically after a set retention period, so it stays useful
without growing forever.

![Action Log](/assets/images/EnvironmentManager/ActionLog.png){: loading="lazy" decoding="async" width="1424" height="253"}

```mermaid
flowchart TB
    CLICK(["Operator picks rows,<br/>chooses an action, sets options"]) --> LOG["Action Log entry created<br/>Status = Pending"]
    LOG --> NOW{"Execute immediately?"}
    NOW -->|no| HOLD["stays Pending<br/>until its scheduled time"]
    HOLD --> DUE(["due: background job<br/>picks it up"])
    NOW -->|yes| BUSY
    DUE --> BUSY{"another operation<br/>in progress on this<br/>environment?"}
    BUSY -->|yes| QUEUE["Status = Queued"]
    BUSY -->|no| VALIDATE{"tenant reachable<br/>+ operation valid?"}
    VALIDATE -->|no| FAIL["Status = Failed<br/>error message stored"]
    VALIDATE -->|yes| SEND["build request,<br/>call Admin Centre API"]
    SEND --> IDRESP{"API returns<br/>operation id?"}
    IDRESP -->|no| FAIL
    IDRESP -->|yes| INPROG["Status = In Progress<br/>operation id stored"]

    INPROG --> POLL(["Action Log poller<br/>runs every minute"])
    POLL --> CHECK["poll operation id<br/>on the Admin Centre API"]
    CHECK --> STATE{"result?"}
    STATE -->|still running| POLL
    STATE -->|succeeded| OK["Status = Completed<br/>affected cache records refreshed"]
    STATE -->|failed| FAIL
    OK --> RELEASE["release next Queued operation<br/>for that environment"]
    RELEASE --> BUSY
```

This is the part I enjoyed building most. The log *is* the state machine. A
long operation returns an ID; a background poller periodically checks that ID and
updates the row. And because the Admin Centre API refuses to run two operations
against the same environment at once, extra requests are not rejected and lost —
they sit as **Queued** and are released one by one as each in-flight operation
finishes. Multi-select stays safe even when every row targets the same
environment.

## A worked example: deploying an extension to many customers

Rolling a product out to a set of customers ties the whole thing together.

1. Select the target environments, choose **Deploy Extension**, and either upload
   an `.app` package or pick products from a configured package feed.
2. On close, the tool reads the package's dependency graph, pulls any missing
   dependencies from the feed, and builds a **batch** ordered so prerequisites
   install first. Anything it cannot resolve becomes a warning, not a dead stop —
   the rest of the batch still goes ahead.
3. A background dispatcher runs the batch **one deployment at a time per
   environment**, uploads each package, then polls the Admin Centre until it
   finishes.
4. Progress is visible on a dedicated **Extension Deployments** page, each row
   moving `Scheduled → Uploading → Installing → Completed`. A failed row keeps its
   error message and a **Retry** action.

![Extension Selection](/assets/images/EnvironmentManager/DeploymentSelection.png){: loading="lazy" decoding="async" width="641" height="391"}

![Extension Deployments](/assets/images/EnvironmentManager/ExtensionDeployments.png){: loading="lazy" decoding="async" width="1797" height="230"}

```mermaid
stateDiagram-v2
    [*] --> Scheduled
    Scheduled --> Uploading: dispatcher picks it up<br/>(one per environment at a time)
    Uploading --> Installing: package sent to pteInstall
    Installing --> Completed: Admin Centre confirms
    Uploading --> Failed
    Installing --> Failed
    Scheduled --> Cancelled: operator cancels
    Scheduled --> Skipped: version already installed
    Failed --> Scheduled: Retry
    Completed --> [*]
    Cancelled --> [*]
    Skipped --> [*]
```

Each deployment is also an entry in the same Action Log as everything else, so it
is audited and retained exactly like a copy or a restore.

## The safety rails

Since I would be the one running bulk operations across real customer tenants, I
wanted the tool to stop invalid operations before they start rather than after
they fail.

- Production environments cannot be casually deleted — the check has to be
  deliberately cleared.
- Restore, copy and recover are only offered on environments that can actually
  accept them.
- *Upgrade app* is hidden when no update exists.
- Package size limits and invalid scheduling combinations are flagged in the
  dialog, with a clear message, not surfaced as an API error minutes later.

If an operation does not apply to the selected row, the button is disabled or the
row is filtered out.


## Why build it this way

The payoff for me was direct: the manual estate work became a few clicks with a
paper trail, and the slow operations now look after themselves while I do
something else.

The more interesting outcome is that, once I stripped the project back, it turned
out to be four ideas that fit almost any multi-tenant admin tool on the platform:

- **A local cache kept fresh by scheduled jobs**, so the UI never waits on a
  remote API.
- **An action log as the operational spine** — one table that is both the audit
  trail and the work queue.
- **A background poller** that turns long-running async operations into a status
  you can watch.
- **Per-environment serialization**, so bulk actions cannot trip over the API's
  own concurrency limits.

None of those are large pieces on their own. Together they turned a chore I had
been doing by hand for months into a console I would happily hand to someone on
their first week — which, for a side project, is about as good an outcome as I
was hoping for.

## Where it could go next

Because the cache already holds a normalised copy of every tenant, environment,
app version, and storage figure, most of the obvious extensions are just more
tables next to the ones already there:

- **Licensing.** Pull subscription and per-user licence assignments from the
  Partner Centre alongside the environment data, so you can see entitlement
  against actual usage per customer — seats paid for versus seats active, add-ons
  assigned to the right tenants, renewals coming up.
- **Reporting and dashboards.** The cache tables are ordinary Business Central
  data, so exposing them as API pages or reading them straight from the database
  gives Power BI (or any reporting tool) an estate-wide dataset with no extra
  plumbing: version spread across customers, storage trends, upgrade compliance,
  operation volumes and failure rates from the action log.
- **Alerting.** The same scheduled jobs that refresh the cache are a natural
  place to raise a notification when a customer crosses a storage threshold, an
  environment falls two versions behind, or a scheduled upgrade fails.

The console is the interactive front end; the cache underneath it is really just
a small data warehouse for your Business Central estate, and that is the part
with the most room to grow.
