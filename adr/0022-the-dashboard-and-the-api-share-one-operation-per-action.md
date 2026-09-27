---
authors: [Theauxm]
repos: [scheduler, api, dashboard]
areas: [platform, graphql]
status: accepted
---

# The dashboard and the API share one operation per action

Every action an operator can take in Trax.Dashboard has a counterpart in the GraphQL
`operations` namespace, and both call the same method underneath: `IOperationsService` or
`ITraxScheduler` in Trax.Scheduler, or the mediator where the action is a train's own
execution. Neither surface carries its own validation, input reading, authorization or
persistence for an action. The GraphQL API is then enough to build another frontend on, the
React dashboard among them, with nothing the Blazor dashboard can do that it cannot.

## Status

**Accepted.**

## Why this is written down

Because the two surfaces kept drifting, and each drift was a bug on one side only. The
dashboard's re-queue deserializes and re-serializes the saved input while the API passes it
through, so the dashboard re-queues a truncated input with default values that the API
refuses. The Run dialog wrote its metadata row before submitting and left it Pending when the
submit failed, read enum inputs one way on one tab and another on the other, and ignored the
submitter a train is routed to. The API had no run at all. None of that is visible from
either surface alone.

## Considered options

**Each surface implements the action itself.** What the code did for re-queue and run. It is
quicker for the first surface and wrong by the second, because nothing makes the copies agree.

**The dashboard calls its own GraphQL API over HTTP.** Rejected. The dashboard is Blazor
Server running in the host's process, often in a host that serves no API, and a circuit has
no HTTP request whose caller an HTTP hop could carry. It would add authentication plumbing to
reach code it can already call.

**Share at the mediator.** Only for execution. The admin state (the work queue, runs,
manifests, dead letters, scheduler settings) belongs to the Scheduler, so the shared method
for everything but a train's own execution lives in `IOperationsService`.

## Consequences

**A new action lands in three places together**: the operations method, the GraphQL field that
calls it, and the dashboard control that calls it. A new API field is also measured at scale in
`Trax.Api.Tests.Stress`, at millions of rows, because another frontend will page and poll it
the way the dashboard does.

**Surfaces differ only in who the caller is.** The dashboard calls inside a trusted scope, gated
as a whole by its host (0017); the API calls as the request's user. The operation decides what
a refusal is and what is thrown (scheduler/0004), so both surfaces report the same failure the
same way.

**Some actions do not comply yet.** When this was recorded, re-queue still had one
implementation per surface, and run had its shared method (`IOperationsService.RunTrainAsync`)
but neither surface calling it: the dashboard switches once a Scheduler release carries it, and
the API's run mutation follows the same release. Those are defects against this decision, not
exceptions to it.

**`RunTrainAsync` copies two mediator rules.** The published mediator keeps its authorization
check and its input reading private, so the run path repeats both, and they can drift until the
mediator exposes them.

**`IOperationsService` grows.** A member added to it is a break for anyone implementing the
interface themselves, so a new member has a default implementation that throws
`NotSupportedException`.

## Exemplars

**Enforced elsewhere:** `OperationsServiceRunTests` in Trax.Scheduler pins the run path the two
surfaces share, and `OperationsServiceEnqueueTests` the queue path. `WorkQueueCreationSitesTests`
in Trax.Dashboard and Trax.Api allows neither surface to build a work queue row itself.

Not covered: nothing checks that a dashboard page calls the operations service rather than a
data context or a job submitter, and nothing checks that each dashboard action has a GraphQL
field. Parity is held by review and by the audit list, not by a guard.

## Changelog

- **2026-09-27**: Recorded.
