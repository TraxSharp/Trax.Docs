---
layout: default
title: IOperationsService
parent: Scheduler API
grand_parent: SDK Reference
nav_order: 14
---

# IOperationsService

The operations the dashboard and the GraphQL `operations` namespace share. Each action an operator can take has one method here, and both surfaces call it, so the dashboard and the API validate, authorize and report failures the same way. Registered as scoped by `AddScheduler(...)`. An API-only host registers it itself with `services.AddScoped<IOperationsService, OperationsService>()` (see [AddTraxGraphQL](/docs/sdk-reference/graphql-api/add-trax-graphql)).

## Signature

```csharp
namespace Trax.Scheduler.Services.Operations;

public interface IOperationsService
{
    Task<OperationResult> QueueTrainAsync(QueueTrainInput input, CancellationToken ct);
    Task<OperationResult> RunTrainAsync(RunTrainInput input, CancellationToken ct);
    Task<OperationResult> CancelWorkQueueEntryAsync(long id, CancellationToken ct);
    Task<OperationResult> UpdateManifestGroupAsync(long id, UpdateManifestGroupInput input, CancellationToken ct);
    Task<OperationResult> UpdateSchedulerConfigAsync(UpdateSchedulerConfigInput input, CancellationToken ct);
    // plus the read operations: metrics, scheduler config, manifest group graphs
}

public record OperationResult(bool Success, long? Id = null, int? Count = null, string? Message = null);
```

## Queueing and running

| Method | What it does | `Id` on success |
|--------|--------------|-----------------|
| `QueueTrainAsync(QueueTrainInput(TrainName, InputJson, Priority, ScheduledAt), ct)` | Enqueues through `ITrainExecutionService.QueueAsync`, so the train's `[TraxAuthorize]` requirements, its `OnQueue` hook and its subject key apply. The entry waits for dispatch like any other. | the work queue entry |
| `RunTrainAsync(RunTrainInput(TrainName, InputJson), ct)` | Writes a `Pending` run and submits it at once to the job submitter the train is routed to, the same routing the job dispatcher uses (`ForTrain<T>()`, then `[TraxRemote]`, then the default submitter). Nothing goes through the work queue. | the run's metadata row |

Both look the train up by its interface `FullName` and read `InputJson` the way the mediator reads a caller's input: the system serializer options with property names matched whatever their case (`customerId`, `CustomerId` and `CUSTOMERID` all fill the same property), a property given twice in any casing refused as invalid input rather than resolved to its last value, the mediator's input size cap, and a blank input read as `{}`, which the input type must be buildable from. `QueueTrainAsync` reads this way once it runs against a Trax.Mediator release that carries the change; until then a queued input's property names are case-sensitive.

A run is a deliberate bypass of the work queue. It skips dispatch priority, group `MaxActiveJobs`, and the subject lock, so it can run while another run for the same subject is in progress (see [QueueSubjectKey](/docs/core/trains-and-junctions#queuesubjectkey-serializing-work-that-touches-the-same-thing)). Use `QueueTrainAsync` when that matters.

## Failures

Both methods return a failed `OperationResult` with a `Message` for an answer the caller can act on, and throw for a fault on the server's side.

| Outcome | `QueueTrainAsync` | `RunTrainAsync` |
|---------|-------------------|-----------------|
| Blank `TrainName`, unknown train | failed result | failed result |
| Invalid, oversized or `null` `InputJson`, or a property given twice | failed result (`Invalid InputJson: ...`) | failed result, same message; no run is written |
| The train's `OnQueue` or `QueueSubjectKey` refused | failed result (`The enqueue was refused: ...`) | not applicable: a run has neither |
| The caller may not run the train | throws `UnauthorizedAccessException` | throws `UnauthorizedAccessException`, before the input is read |
| `[TraxAuthorize]` train, no enforcer, not trusted | failed result (`The enqueue was refused: ...`) | throws `InvalidOperationException` |
| Database or network failure | logged and thrown | logged and thrown |
| The job submitter failed | not applicable | the run is marked `Failed` with the submitter's exception, then it is logged and thrown |
| `ct` cancelled | throws `OperationCanceledException` | throws `OperationCanceledException`; a run cancelled before its submit completed is marked `Failed` |

A thrown failure reaches Trax.Api's error filter, which masks any type it does not know as `Unexpected Execution Error`, so a connection string's host and port never reach a GraphQL client.

## Authorization

Neither method decides authorization itself: `QueueTrainAsync` leaves it to the mediator, and `RunTrainAsync` applies the mediator's rule. An `ITrainAuthorizationService` decides when one is registered (Trax.Api registers one). Without one, a call inside a trusted scope passes, and a `[TraxAuthorize]` train is refused unless the host called `AllowMissingAuthorizationService()`. The dashboard calls both inside the `"dashboard"` trusted scope, because it is gated as a whole by its host (see [Authorization: The Operations Surface](/docs/authorization#the-operations-surface)).

## Implementing it yourself

`RunTrainAsync` has a default implementation that throws `NotSupportedException`, so an implementation written before it was added still compiles. `OperationsService` needs the constructor that takes an `IServiceProvider`, which dependency injection picks, to resolve a job submitter; built with the older constructor, its `RunTrainAsync` throws `InvalidOperationException`.

## Package

```
dotnet add package Trax.Scheduler
```
