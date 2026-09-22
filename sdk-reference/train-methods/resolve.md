---
layout: default
title: Resolve
parent: Train Methods
grand_parent: SDK Reference
nav_order: 7
---

# Resolve

Ends a chain, taking the train's `TReturn` out of Memory. Follows a priority chain: exception >
short-circuit value > Memory lookup.

Every chain ends with it. On a chain that names junctions it is the last call in the fluent
sequence; on a chain that names none, because the train's return type is already in Memory as the
input type or as `Unit`, it is the only call.

## Signature

```csharp
protected Either<Exception, TReturn> Resolve()
```

There is no overload taking a value. A declaration says which junctions run, so stating a result
directly would let it return something the junctions it names never produced.

## Returns

`Either<Exception, TReturn>`, the train result. `Left` contains the exception on failure; `Right`
contains the `TReturn` value on success.

## Examples

### Ending a chain of junctions

```csharp
protected override Task<Either<Exception, OrderResult>> Junctions() =>
    Chain<ValidateOrder>()
        .Chain<ProcessPayment>()   // Stores OrderResult in Memory
        .Resolve();                // Takes OrderResult back out
```

### A chain that names no junctions

The train's return type is already in Memory, so there is nothing to chain.

```csharp
public class EchoTrain : ServiceTrain<string, string>, IEchoTrain
{
    protected override Task<Either<Exception, string>> Junctions() =>
        Task.FromResult(Resolve());
}
```

## Remarks

- `Resolve()` does not throw. It returns an `Either`. The calling `Run()` method unwraps the Either and throws if needed.
- The parameterized `Resolve(returnType)` is simpler: it returns the exception if one exists, otherwise returns the provided value. It does **not** check short-circuit or Memory.
