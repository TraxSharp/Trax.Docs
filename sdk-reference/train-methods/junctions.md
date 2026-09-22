---
layout: default
title: Junctions
parent: Train Methods
grand_parent: SDK Reference
nav_order: 0
---

# Junctions

Override `Junctions()` to define the train's route, the sequence of junctions it passes through. This is the primary way to compose junctions in a train.

## Signature

```csharp
// Train<TInput, TReturn>
protected virtual TReturn Junctions()

// ServiceTrain<TIn, TOut>: same signature
protected virtual TOut Junctions()
```

## Returns

`TReturn`, the train's output type. The return value is resolved automatically from Memory via an implicit conversion on `Monad`. You do not need to call `Resolve()` or wrap the result in `Either`.

## Examples

### Basic Train

```csharp
public class CreateUserTrain : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override Task<Either<Exception, User>> Junctions() =>
        Chain<ValidateEmailJunction>()
            .Chain<CreateUserJunction>().Resolve();
}
```

### With ShortCircuit and Extract

All chain methods (`Chain`, `ShortCircuit`, `Extract`, `AddServices`) are available as protected methods on the train:

```csharp
public class ProcessOrderTrain : ServiceTrain<OrderInput, OrderResult>
{
    protected override OrderResult Junctions() =>
        ShortCircuit<CheckCacheJunction>()
            .Chain<ValidateOrderJunction>()
            .Extract<OrderInput, OrderDetails>()
            .Chain<ProcessPaymentJunction>();
}
```

### With AddServices

```csharp
public class NotifyTrain(ISlackClient slack) : ServiceTrain<NotifyInput, Unit>
{
    protected override Unit Junctions() =>
        AddServices<ISlackClient>(slack)
            .Chain<SendNotificationJunction>();
}
```

## Behavior

1. The framework calls `Activate(input)` automatically before `Junctions()` executes, seeding Memory with the train input and `Unit`.
2. Chain methods are called as protected methods on the train itself (not on a separate `Monad` returned by `Activate`).
3. The final chain call returns a `Monad<TInput, TReturn>`, which is implicitly converted to `TReturn` by extracting the result from Memory.
4. If any junction threw an exception, the implicit conversion returns `default(TReturn)` and the framework handles the exception via the railway error path.
5. The `Run()` / `RunEither()` public API is unchanged for callers.

## There is no alternative to Junctions()

`RunInternal` is private and `Activate` is internal, so a chain cannot be assembled in code. A
chain built imperatively has no single shape, which would put it out of reach of the check a host
runs over every train before it serves traffic.

Everything that used to justify reaching for `RunInternal` has a place in the chain:

| What you need | Where it goes |
|---|---|
| Logic before or after the chain | A junction at the head or the tail of it |
| Extra objects in Memory | A junction whose return value is that object |
| Returning a failure | A junction throws; the chain turns it into `Left` |
| Async setup | A junction, which is async already |
| Merging a nested train's result | A junction that calls `TrainBus` and returns the merged value |

```csharp
public class ParentTrain : ServiceTrain<ParentInput, ParentResult>, IParentTrain
{
    protected override Task<Either<Exception, ParentResult>> Junctions() =>
        Chain<RunChildTrain>().Chain<ValidateJunction>().Resolve();
}

internal class RunChildTrain(ITrainBus trainBus) : Junction<ParentInput, ParentResult>
{
    public override async Task<ParentResult> Run(ParentInput input)
    {
        var childResult = await trainBus.RunAsync<ChildResult>(
            new ChildRequest { Data = input.ChildData }, CancellationToken);

        return new ParentResult { ParentData = input.ParentData, ChildResult = childResult };
    }
}
```

## Remarks

- The implicit conversion from `Monad<TInput, TReturn>` to `TReturn` calls `Resolve()` internally, following the same resolution priority: exception > short-circuit value > Memory lookup.
- `TrainInput` and `TrainOutput` throw while the chain is being declared. A chain that branched on its input would have no single shape, so the shape checked at startup need not be the one that runs.
