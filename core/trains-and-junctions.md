---
layout: default
title: Trains & Junctions
parent: Core
nav_order: 1
---

# Trains & Junctions

## Junctions

Junctions are the points along a train's route. Each one does one thing:

```csharp
public class ValidateEmailJunction(IUserRepository UserRepository) : Junction<CreateUserRequest, Unit>
{
    public override async Task<Unit> Run(CreateUserRequest input)
    {
        var existingUser = await UserRepository.GetByEmailAsync(input.Email);
        if (existingUser != null)
            throw new ValidationException($"Email {input.Email} already exists");

        return Unit.Default;
    }
}

public class CreateUserJunction(IUserRepository UserRepository) : Junction<CreateUserRequest, User>
{
    public override async Task<User> Run(CreateUserRequest input)
    {
        var user = new User
        {
            Email = input.Email,
            FullName = $"{input.FirstName} {input.LastName}",
            CreatedAt = DateTime.UtcNow
        };

        return await UserRepository.CreateAsync(user);
    }
}
```

Junctions use constructor injection for dependencies. When a junction throws, the train switches to the left track and returns the exception.

## CancellationToken in Junctions

Every junction has a `CancellationToken` property that is set automatically by the train before `Run()` is called. Use it to pass cancellation to async operations:

```csharp
public class FetchUserJunction(IHttpClientFactory httpFactory) : Junction<UserId, UserProfile>
{
    public override async Task<UserProfile> Run(UserId input)
    {
        var client = httpFactory.CreateClient();
        var response = await client.GetAsync($"/users/{input.Value}", CancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<UserProfile>(CancellationToken);
    }
}
```

The token comes from the caller: `train.Run(input, cancellationToken)`. If no token is provided, `CancellationToken` defaults to `CancellationToken.None`. Before each junction executes, cancellation is checked. If the token is already cancelled, the junction is skipped and `OperationCanceledException` propagates.

*Full details: [Cancellation Tokens](/docs/cross-cutting/cancellation-tokens)*

## EffectJunction vs Junction

Trax has two junction base classes:

**`Junction<TIn, TOut>`** is the base class. Handles input/output and railway error propagation. No metadata, no lifecycle hooks. Use this for lightweight junctions or when running inside a plain `Train`.

**`EffectJunction<TIn, TOut>`** extends `Junction` with per-junction metadata tracking. When run inside a `ServiceTrain`, it records a `JunctionMetadata` entry with the junction's name, input/output types, start/end times, and railway state. Junction effect providers (like `AddJunctionLogger`) hook into `EffectJunction`'s lifecycle and fire before and after each junction executes.

```csharp
// Base junction, no metadata tracking
public class ValidateEmailJunction(IUserRepository repo) : Junction<CreateUserRequest, Unit>
{
    public override async Task<Unit> Run(CreateUserRequest input)
    {
        if (await repo.GetByEmailAsync(input.Email) != null)
            throw new ValidationException("Email already exists");
        return Unit.Default;
    }
}

// Effect junction, tracked by junction effect providers
public class ValidateEmailJunction(IUserRepository repo) : EffectJunction<CreateUserRequest, Unit>
{
    public override async Task<Unit> Run(CreateUserRequest input)
    {
        if (await repo.GetByEmailAsync(input.Email) != null)
            throw new ValidationException("Email already exists");
        return Unit.Default;
    }
}
```

The implementation is identical. Just swap the base class. `EffectJunction` only adds metadata when running inside a `ServiceTrain`. If you use `EffectJunction` inside a plain `Train`, it throws at runtime.

Use `EffectJunction` when you want junction-level observability (timing, logging via `AddJunctionLogger`). Use `Junction` when you don't need it.

## Dependency Injection in Junctions

Junctions use standard constructor injection for their dependencies. Do **not** use the `[Inject]` attribute. That's used internally by the `ServiceTrain` base class for its own framework-level services.

```csharp
// Don't use [Inject] in your junctions
public class MyJunction : Junction<Input, Output>
{
    [Inject]
    public IMyService MyService { get; set; }
}

// Use constructor injection
public class MyJunction(IMyService MyService) : Junction<Input, Output>
{
    public override async Task<Output> Run(Input input)
    {
        var result = await MyService.DoSomethingAsync(input);
        return result;
    }
}
```

## Train Structure

As your application grows, you'll want a consistent way to organize trains. Group each train with its input, interface, and junctions in a single folder:

```
Trains/
├── CreateUser/
│   ├── CreateUserRequest.cs        # Input model
│   ├── ICreateUserTrain.cs         # Interface
│   ├── CreateUserTrain.cs          # Implementation
│   └── Junctions/
│       ├── ValidateEmailJunction.cs
│       ├── CreateUserInDatabaseJunction.cs
│       └── SendWelcomeEmailJunction.cs
│
├── ProcessOrder/
│   ├── ProcessOrderRequest.cs
│   ├── IProcessOrderTrain.cs
│   ├── ProcessOrderTrain.cs
│   └── Junctions/
│       ├── ValidateOrderJunction.cs
│       ├── ChargePaymentJunction.cs
│       └── CreateShipmentJunction.cs
```

### The Input Model

Each train gets its own request type. This is required by the `TrainBus` because input types must be unique across your application:

```csharp
namespace YourApp.Trains.CreateUser;

public record CreateUserRequest
{
    public required string Email { get; init; }
    public required string FirstName { get; init; }
    public required string LastName { get; init; }
}
```

### The Interface

Define an interface for DI resolution and testing:

```csharp
namespace YourApp.Trains.CreateUser;

public interface ICreateUserTrain : IServiceTrain<CreateUserRequest, User>;
```

### The Junctions Folder

Junctions go in a `Junctions/` subfolder. Mark them `internal` since they're implementation details of this train:

```csharp
namespace YourApp.Trains.CreateUser.Junctions;

internal class ValidateEmailJunction(IUserRepository userRepository) : Junction<CreateUserRequest, Unit>
{
    public override async Task<Unit> Run(CreateUserRequest input)
    {
        var existing = await userRepository.GetByEmailAsync(input.Email);
        if (existing != null)
            throw new ValidationException($"Email {input.Email} already exists");

        return Unit.Default;
    }
}
```

Using `internal` keeps your public API surface clean. External code interacts with `ICreateUserTrain`, not individual junctions.

### When to Share Junctions

Sometimes multiple trains need the same validation or transformation. Resist the urge to share junctions too early. Duplication is often cheaper than the wrong abstraction.

When you do need to share, create a `Shared/` folder at the `Trains/` level:

```
Trains/
├── Shared/
│   └── Junctions/
│       └── ValidateEmailFormatJunction.cs
├── CreateUser/
│   └── ...
├── UpdateUser/
│   └── ...
```

Shared junctions should be truly generic. If you find yourself adding conditionals to handle different trains, that's a sign the junction should be duplicated and specialized instead.

## Defining a Train

Override `Junctions()` to define the route, the sequence of junctions the train passes through:

```csharp
public class CreateUserTrain : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override Task<Either<Exception, User>> Junctions() =>
        Chain<ValidateEmailJunction>()
            .Chain<CreateUserJunction>().Resolve();
}
```

`Junctions()` returns `TReturn` directly. There is no `Either`, no `async Task`, no `Activate`, no `Resolve`. The framework handles all of that. Chain methods (`Chain`, `ShortCircuit`, `Extract`, `AddServices`) are available as protected methods on the train itself.

### There is no escape hatch, on purpose

`Junctions()` is the only way to declare a chain. `RunInternal` is private and `Activate` is
internal, so a train cannot build its chain imperatively.

That is what makes a chain readable before it runs. A chain assembled in code has no single
shape, so the host could not check it at startup, and a chain that varies by input would mean the
shape verified is not necessarily the shape that runs. To keep the guarantee, the declaration has
to be the only option.

Two rules follow from it:

**A chain cannot read the value being processed.** `TrainInput` and `TrainOutput` throw while a
chain is being declared. Work that needs the input belongs in a junction, which receives it as
its argument.

**A chain cannot seed arbitrary values into Memory.** A value a later junction reads is produced
by an earlier junction. That is the same mechanism in both directions: a junction's return value
lands in Memory keyed by its output type.

```csharp
public class CreateUserTrain : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override Task<Either<Exception, User>> Junctions() =>
        Chain<StampCorrelationId>()   // work that used to sit above the chain
            .Chain<ValidateEmailJunction>()
            .Chain<CreateUserJunction>()
            .Resolve();
}
```

`Resolve()` on its own ends a chain that declares no junctions, for a train whose return type is
already in Memory because it is the input type or `Unit`.

### The host checks every chain before it serves traffic

Because a chain is a declaration of types, whether it can run is decidable without running it. At
startup Trax reads every registered train's chain and refuses to start if one of them cannot run:

- the chain could not be read, because it reads the input
- it names a junction that is neither registered nor constructible
- a junction needs something in Memory that nothing before it produces
- the chain ends without the train's return type in Memory

Every train is checked before anything is reported, so one start tells you about all of them.

#### Reading the input inside `Junctions()`

The first of those is the one most likely to bite on upgrade. `Junctions()` declares a chain; it
does not process a value, so there is no input to read yet. `TrainInput` and `TrainOutput` throw
`ChainDeclarationException` while a chain is being declared, and the startup check turns that into
a refused start.

```csharp
protected override Task<Either<Exception, Unit>> Junctions()
{
    // Throws: there is no input at declaration time.
    if (!string.IsNullOrEmpty(TrainInput.ExternalId))
        this.ExternalId = TrainInput.ExternalId;

    return Chain<DoWorkJunction>().Resolve();
}
```

Work that needs the input belongs in a junction, which receives it, or — for something that must
happen as the mutation is accepted — in [`OnQueue`](#onqueue-enqueue-time-hook), which is handed a
`Metadata` carrying the input.

This is a compile-clean change that only surfaces at startup, so a consumer upgrading has no way to
discover it beforehand. If an upgrade is blocked on it, `SkipChainVerification()` turns the check
off while the chains are moved over — but it silences every other fault in the list too, so it is a
stopgap rather than a setting to leave on.

The replay knows the types a chain declares, not the concrete types that will flow, so a junction
declaring an interface that its runtime value implements only incidentally reads as a fault. That
is the one case for turning the check off:

```csharp
.AddMediator(mediator => mediator.SkipChainVerification())
```

## Train Lifecycle Hooks

`ServiceTrain` provides `protected virtual` methods you can override to react to your train's own lifecycle events, with no global hook registration needed:

```csharp
public class CreateUserTrain(ISlackClient slack)
    : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override User Junctions() =>
        Chain<ValidateEmailJunction>()
            .Chain<CreateUserJunction>();

    protected override async Task OnFailed(
        Metadata metadata, Exception exception, CancellationToken ct)
    {
        await slack.PostAsync($"User creation failed: {exception.Message}", ct);
    }
}
```

Available overrides: `OnStarted`, `OnCompleted`, `OnFailed`, `OnCancelled`. All default to no-op. Exceptions in overrides are caught and logged and never cause the train to fail. Each hook receives the metadata populated with everything known at that point: `OnStarted` sees the typed input (via `TrainInput` or `metadata.GetInput<T>()`); `OnCompleted` additionally sees the output; `OnFailed`/`OnCancelled` see the failure state.

These work alongside [global lifecycle hooks](/docs/sdk-reference/configuration/add-lifecycle-hook). Global hooks fire first, then per-train overrides.

### OnQueue (enqueue-time hook)

`OnQueue` is a fifth override that fires at a different moment from the others: synchronously inside the mediator's QUEUE path (`ITrainExecutionService.QueueAsync`), **before** the work queue row is inserted. The other four fire when the train *runs*; `OnQueue` fires when a QUEUE mutation is *accepted*, before the train is scheduled. It does not fire on the synchronous RUN path, and it does not fire again when the scheduler later runs the train.

Use it to perform a side-effect the instant a mutation is accepted, instead of waiting for the deferred run. A common case is an optimistic write: persist a provisional row so the change is visible immediately, then let the real run reconcile it.

```csharp
public class ProcessMatchResultTrain
    : ServiceTrain<ProcessMatchResultInput, ProcessMatchResultOutput>, IProcessMatchResultTrain
{
    [Inject]
    public IDbContextFactory<GameDbContext>? GameDbFactory { get; set; }

    protected override async Task OnQueue(Metadata metadata, CancellationToken ct)
    {
        var input = metadata.GetInput<ProcessMatchResultInput>();
        if (input is null || GameDbFactory is null)
            return;

        await using var db = await GameDbFactory.CreateDbContextAsync(ct);
        db.Matches.Add(new MatchRecord { MatchId = input.MatchId, Region = input.Region });
        await db.SaveChangesAsync(ct);
    }
}
```

`OnQueue` differs from the other four hooks in three ways:

- **Exceptions propagate.** A throw is not swallowed: it aborts the enqueue and no work queue row is written. Use it only for work that must succeed for the mutation to be accepted.
- **The train is not initialized.** `this.Metadata` and `TrainInput` are unavailable. Read everything from the passed `metadata`: the input via `metadata.GetInput<T>()`, and `metadata.ExternalId` to correlate with the eventual run, which executes under the same ExternalId. `Id`, `ManifestId`, and `ScheduledTime` are unset because no run exists yet.
- **It must be idempotent.** The deferred run re-executes the full `Junctions()` chain, so any effect the chain also performs will happen again. Write `OnQueue` so running it plus the chain is safe.

Property dependencies marked `[Inject]` (like `GameDbFactory` above) are populated before `OnQueue` is called, the same as during a normal run. Trains that do not override `OnQueue` skip resolution entirely, so the enqueue path is unaffected.

#### Making the side-effect durable

The hook and the work queue row are two writes. If the process dies between them, the side-effect can be left with no queued work to consume it — a provisional row nothing will ever reconcile. There are two ways to close that, and which one applies depends on where the hook writes.

**Writing through Trax's own context.** `IEnqueueContextAccessor.Current` exposes the data context the enqueue is about to commit on. Anything tracked on it is committed with the work queue row and rolled back with it:

```csharp
protected override async Task OnQueue(Metadata metadata, CancellationToken ct)
{
    await accessor.Current!.Track(someTraxEntity);   // no SaveChanges — the enqueue commits it
}
```

`Current` is non-null only while `OnQueue` is running, and the hook must not call `SaveChanges` or commit on it: the enqueue owns the lifetime. This only covers entities in Trax's own model.

**Writing through your own `DbContext`.** EF can only share a transaction between contexts that share a connection, so a separately-registered context — the common case, and the one in the example above — commits independently and cannot be rolled back with the entry. For that, defer promotion:

```csharp
protected override bool DeferQueuePromotion => true;
```

The entry is then committed **unconfirmed** and is not dispatchable. The hook runs. A second commit stamps `confirmed_at` and the entry becomes claimable. This does not make the two writes atomic — nothing can, across two databases — but it makes a failure *findable*: a crash leaves an unconfirmed entry instead of an invisible side-effect. `IWorkQueuePromotion.PromoteStaleAsync` sweeps those up, promoting rather than cancelling them, because the deferred run re-executes the whole chain and the hook is required to be idempotent anyway.

A hook that *throws* still aborts the enqueue outright: the staged entry is removed, so the observable contract is unchanged either way.

Deferral is opt-in because it costs an extra round trip and earns nothing when the hook writes nowhere Trax's transaction cannot reach. **Immediate promotion is the default**, and a train that does not override `OnQueue` never stages anything.

### QueueSubjectKey (serializing work that touches the same thing)

Queued entries are claimed by several workers at once, keyed on nothing, so two mutations for the same record can run simultaneously. Against a system that resolves concurrent writes by last-write-wins, that is a lost update.

A train can say what its work touches:

```csharp
protected override string? QueueSubjectKey(Metadata metadata) =>
    $"customer-{metadata.GetInput<PatchCustomerInput>()!.CustomerId}";
```

The key is an opaque string — Trax compares it and nothing else, so its shape is yours to choose. A record identity is the usual pick. It is read at enqueue time from a metadata carrying the input, so it varies per mutation rather than being fixed per train.

Returning null, which is the default, means no serialization. Every train that does not override this is unaffected.

**Throwing aborts the enqueue.** A key that cannot be computed must not quietly become null: that would drop the guarantee at exactly the moment the caller was relying on it.

Two limits worth knowing. Only entries created through the mediator's queue path carry a key — work queued from a manifest is not about a record and has no subject, and the dashboard's rerun builds its entry directly rather than through `QueueAsync`. And ordering within a subject is enqueue order *at equal priority*; a higher-priority entry for the same subject still goes first, because priority should mean something.

> The key is recorded on the entry today. The dispatcher change that acts on it — refusing to claim an entry whose subject already has a run in flight — is a separate change; until it lands, the column is carried and not enforced.

## SDK Reference

> [Junctions](/docs/sdk-reference/train-methods/junctions) | [Chain](/docs/sdk-reference/train-methods/chain) | [Resolve](/docs/sdk-reference/train-methods/resolve)
