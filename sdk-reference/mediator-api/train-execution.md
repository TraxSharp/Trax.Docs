---
layout: default
title: TrainExecution
parent: Mediator API
grand_parent: SDK Reference
nav_order: 4
---

# TrainExecution

`ITrainExecutionService` provides programmatic train execution by name. Instead of resolving a specific train interface, you pass the train's service type name and a JSON string. The service handles discovery, deserialization, and dispatch. Train lookup matches by fully-qualified canonical name first (`ServiceType.FullName`), then friendly name (`ServiceTypeName`), then short name (`ServiceType.Name`).

It supports two execution paths:
- **Queue**: creates a WorkQueue entry for asynchronous dispatch by the scheduler.
- **Run**: executes the train synchronously via `ITrainBus` on the current machine.

Registered automatically by `AddMediator()` as a scoped service.

## ITrainExecutionService

```csharp
public interface ITrainExecutionService
{
    Task<QueueTrainResult> QueueAsync(
        string trainName,
        string? inputJson,
        int priority = 0,
        DateTime? scheduledAt = null,
        CancellationToken ct = default
    );

    Task<RunTrainResult> RunAsync(
        string trainName,
        string inputJson,
        CancellationToken ct = default
    );
}
```

## QueueAsync

Creates a WorkQueue entry for asynchronous execution. The scheduler picks up the entry on its next polling cycle and dispatches the train.

```csharp
Task<QueueTrainResult> QueueAsync(
    string trainName,
    string? inputJson,
    int priority = 0,
    DateTime? scheduledAt = null,
    CancellationToken ct = default
)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `trainName` | `string` | Yes | N/A | Train name, matched by canonical name (`ServiceType.FullName`), then friendly name (`ServiceTypeName`), then short name (`ServiceType.Name`). Prefer the fully-qualified interface name (e.g. `"MyApp.Trains.IProcessOrderTrain"`). |
| `inputJson` | `string?` | Yes | N/A | JSON-serialized input matching the train's `InputType`. May be null. |
| `priority` | `int` | No | `0` | Dispatch priority (0-31, higher runs first) |
| `scheduledAt` | `DateTime?` | No | `null` | Earliest time the entry may be dispatched. Stored and compared as UTC, so pass a UTC value. Null dispatches as soon as a worker is free. |
| `ct` | `CancellationToken` | No | `default` | Cancellation token |

**Returns**: `QueueTrainResult`

| Property | Type | Description |
|----------|------|-------------|
| `WorkQueueId` | `long` | Database ID of the created WorkQueue entry |
| `ExternalId` | `string` | External ID assigned to the entry |

**Throws**:
- `TrainNotFoundException` (an `InvalidOperationException`) if no train is registered with the given name. Use `ITrainDiscoveryService.DiscoverTrains()` to list available trains.
- `AmbiguousTrainNameException` if the name matches more than one train's friendly name.
- `TrainInputValidationException` if `inputJson` exceeds the configured size cap (`WithMaxInputJsonBytes`, 256 KiB by default).
- `InvalidOperationException` if JSON deserialization returns null.
- `TrainAuthorizationException` if the train has `[TraxAuthorize]` requirements the caller does not meet. Authorization applies to **every** enqueue, including the operations surface (`queueTrain`, `requeueExecution`) and the dashboard's re-queue, which all route through this method.
- `InvalidOperationException` if the train declares `[TraxAuthorize]` and no `ITrainAuthorizationService` is registered. The check fails closed; a host that serves no API submissions opts out with `AddMediator(m => m.AllowMissingAuthorizationService())`, after which the missing service is a no-op.
- Any exception thrown by the train's `QueueSubjectKey` override. A key that cannot be computed aborts the enqueue rather than becoming null.
- Any exception thrown by the train's [`OnQueue`](/docs/core/trains-and-junctions#onqueue-enqueue-time-hook) hook, if the train overrides it. The hook fires before the entry is persisted, so a throw aborts the enqueue and no entry is written, including when the train defers promotion, where the staged entry is removed.

### What it does

1. Looks up the train by `trainName` via `ITrainDiscoveryService`.
2. Authorizes the caller against the train's requirements, failing closed as described under **Throws**.
3. Deserializes `inputJson` to the train's `InputType`.
4. Re-serializes the input using manifest serialization options (normalizes the JSON).
5. Creates a `WorkQueue` entry with the train name, serialized input, input type name, priority, and `scheduledAt`.
6. Stamps the entry's subject key from the train's `QueueSubjectKey` override, if it has one. An exception from `QueueSubjectKey` propagates and aborts the enqueue, so no entry is written.
7. Tracks the entry, then (if the train overrides [`OnQueue`](/docs/core/trains-and-junctions#onqueue-enqueue-time-hook)) invokes the hook with the entry's `ExternalId` and the input, then saves and commits, all in one transaction. A throw rolls the whole thing back, so nothing the hook tracked on `IEnqueueContextAccessor.Current` survives either. Trains that do not override the hook are never resolved here.
8. Returns the entry's ID and external ID.

Tracking the entry before the hook runs does not insert it (`Track` is change tracking only), so the hook still runs before the row exists, as its contract states.

When the train sets [`DeferQueuePromotion`](/docs/core/trains-and-junctions#making-the-side-effect-durable), the shape changes to three steps instead: the entry is committed **unconfirmed** and undispatchable, the hook runs outside that transaction, and a second commit promotes it. A throwing hook removes the staged entry, so the observable contract is the same. A crash leaves the entry unconfirmed for `IWorkQueuePromotion.PromoteStaleAsync` to recover.

Providers without transaction support (the in-memory provider) degrade to a single `SaveChanges` with no explicit transaction.

## RunAsync

Executes a train synchronously via `ITrainBus`. This is a blocking call that returns when the train completes.

```csharp
Task<RunTrainResult> RunAsync(
    string trainName,
    string inputJson,
    CancellationToken ct = default
)
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `trainName` | `string` | Yes | N/A | Train name (matched by canonical name, then friendly name, then short name) |
| `inputJson` | `string` | Yes | N/A | JSON-serialized input matching the train's `InputType` |
| `ct` | `CancellationToken` | No | `default` | Cancellation token forwarded to `ITrainBus.RunAsync` |

**Returns**: `RunTrainResult`

| Property | Type | Description |
|----------|------|-------------|
| `MetadataId` | `long` | Database ID of the metadata record for this execution |
| `Output` | `object?` | The train's typed output. `null` for `Unit` trains; the actual output object for trains with a typed `TOut` parameter. |

**Throws**:
- `TrainNotFoundException` (an `InvalidOperationException`) if no train is registered with the given name, or `AmbiguousTrainNameException` if the name matches more than one train's friendly name.
- `TrainInputValidationException` if `inputJson` exceeds the configured size cap.
- `InvalidOperationException` if JSON deserialization returns null.
- `TrainException` if the train itself fails during execution (propagated from `ITrainBus`).
- `TrainAuthorizationException` if the train has `[TraxAuthorize]` requirements the caller does not meet.
- `InvalidOperationException` if the train declares `[TraxAuthorize]` and no `ITrainAuthorizationService` is registered, unless the host called `AllowMissingAuthorizationService()`. The same fail-closed rule as `QueueAsync`.

### What it does

1. Looks up the train by `trainName` via `ITrainDiscoveryService`.
2. Authorizes the caller against the train's requirements, failing closed as described under **Throws**.
3. Deserializes `inputJson` to the train's `InputType`.
4. Creates a `Metadata` record with a generated external ID.
5. Persists the metadata via the data context.
6. Calls the typed `ITrainBus.RunAsync<TOut>(input, ct, metadata)` via reflection, using the train's `OutputType` from its registration.
7. Returns the metadata ID and the train's output (or `null` for `Unit` trains).

## Examples

### Queue a train for async dispatch

```csharp
public class OrderController(ITrainExecutionService execution) : ControllerBase
{
    [HttpPost("orders/queue")]
    public async Task<IActionResult> QueueOrder(
        [FromBody] JsonElement input,
        CancellationToken ct)
    {
        var result = await execution.QueueAsync(
            "MyApp.Trains.IProcessOrderTrain",
            input.GetRawText(),
            priority: 5,
            ct);

        return Accepted(new { result.WorkQueueId, result.ExternalId });
    }
}
```

### Run a train synchronously

```csharp
public class OrderController(ITrainExecutionService execution) : ControllerBase
{
    [HttpPost("orders/run")]
    public async Task<IActionResult> RunOrder(
        [FromBody] JsonElement input,
        CancellationToken ct)
    {
        var result = await execution.RunAsync(
            "MyApp.Trains.IProcessOrderTrain",
            input.GetRawText(),
            ct);

        return Ok(new { result.MetadataId });
    }
}
```

### Discover available trains first

```csharp
public class TrainController(
    ITrainDiscoveryService discovery,
    ITrainExecutionService execution
) : ControllerBase
{
    [HttpPost("trains/{trainName}/run")]
    public async Task<IActionResult> RunByName(
        string trainName,
        [FromBody] JsonElement input,
        CancellationToken ct)
    {
        // Validate the train exists before attempting execution
        var trains = discovery.DiscoverTrains();
        var match = trains.FirstOrDefault(t => t.ServiceType.FullName == trainName);

        if (match is null)
            return NotFound($"No train registered with name '{trainName}'");

        var result = await execution.RunAsync(trainName, input.GetRawText(), ct);
        return Ok(new { result.MetadataId });
    }
}
```

## Concurrency Limiting

`RunAsync` supports per-train and global concurrency limits to prevent overloading remote backends. When a limit is reached, additional requests wait in-process until a slot opens. No requests are rejected.

See [Concurrency Limiting](/docs/sdk-reference/mediator-api/concurrency-limiting) for configuration details.

## Package

```
dotnet add package Trax.Mediator
```
