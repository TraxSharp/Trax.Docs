---
authors: [Theauxm]
repos: [mediator, scheduler, api, dashboard]
areas: [platform, graphql]
status: accepted
---

# A caller's enqueue goes through the mediator

Anything that enqueues a train with a train and input a caller chose goes through
`ITrainExecutionService.QueueAsync`, which applies the train's `[TraxAuthorize]` requirements,
its `OnQueue` hook and its subject key. Only enqueues whose train and input a manifest fixed at
startup may build a work queue row themselves, and those are governed by the admin surface's
gate as a whole rather than by each train's requirements.

## Status

**Accepted.**

## Considered options

**Authorizing each surface where it enqueues.** What the code did: the operations service,
the queue dialog and the re-queue button each wrote their own row, and only some of them
checked anything. Every new surface was a new chance to forget, and three of them had. Routing
through one method makes the check impossible to skip by writing a new caller.

**Applying per-train requirements to manifest triggers and dead-letter requeues too.** Rejected
for now. Those paths let the caller choose which manifest, never the train or the input, and
they are reachable only through the admin surface. The admin surface is meant to carry admin
privileges as a whole, and is expected to gain user-provided authorization (Microsoft Entra or
another provider) rather than a per-train check bolted onto each admin action. Until then its
posture is `GateOperations`, `RequireAuthorization` or `AllowAnonymousOperations` (api/0004).

**Checking authorization inside `TraxScheduler`.** Rejected because `TraxScheduler` is also
called from a host's own background code, where there is no user; a check there would refuse
every such call for a train with requirements.

## Consequences

The operations service authorizes before it reads the caller's input, so a caller who may not
run a train learns nothing about the input it expects from a parse error. `OperationsService`
takes `ITrainExecutionService` as a required dependency, so an API-only host needs
`AddMediator` as well.

## Exemplars

**Enforced elsewhere:** `WorkQueueCreationSitesTests` in the `Tests.Meta` projects of
Trax.Scheduler (allow-list: the ManifestManager's scheduled enqueue, dormant dependents, and
`TraxScheduler`'s manifest trigger and dead-letter requeue), Trax.Api and Trax.Dashboard (no
allowed sites). `OperationsServiceAuthorizationTests` in Trax.Scheduler runs the operations
service against the real mediator, and `QueueTrainAuthorizationTests` in Trax.Api pins the
`TRAX_AUTHORIZATION` error `queueTrain` returns.

Not covered: the guards find `WorkQueue.Create` by text, so a row built another way, or through
a helper outside `src`, would pass. Nothing checks that the manifest paths stay behind the admin
gate; that posture is api/0004's.

## Changelog

- **2026-09-23**: Recorded.
