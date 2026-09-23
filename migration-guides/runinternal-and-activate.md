# Removal of RunInternal and Activate

`Junctions()` is now the only way to declare a train's chain. `Train.RunInternal` is private,
`Train.Activate` is internal, and `ServiceTrain.Activate` is gone. A train that overrode
`RunInternal` or called `Activate` no longer compiles. The reason is in
[ADR 0016](/docs/adr/0016-a-junction-chain-is-a-declaration-not-a-step-of-the-work): a chain built
imperatively has no single shape, so the host could not read it at startup.

## Overriding RunInternal

Before:

```csharp
protected override Task<Either<Exception, TOut>> RunInternal(TIn input) =>
    Activate(input).Chain<A>().Resolve();
```

After:

```csharp
protected override Task<Either<Exception, TOut>> Junctions() =>
    Chain<A>().Resolve();
```

The framework seeds Memory with the input before `Junctions()` runs, so the chain starts at the
first junction. A chain that names no junctions, because the return type is already in Memory as
the input type or `Unit`, becomes `Task.FromResult(Resolve())`.

## Code that read the input above the chain

`Junctions()` declares a chain; it does not process a value. `TrainInput` and `TrainOutput` throw
`ChainDeclarationException` while the chain is being read, and the startup check turns that into a
refused start. Code that used the `input` parameter of `RunInternal` moves to one of two places:

| The work | Where it goes |
|---|---|
| Needs the input as part of the run | A junction at the head of the chain, which receives the input as its argument |
| Must happen the moment a queued mutation is accepted | [`OnQueue`](/docs/core/trains-and-junctions#onqueue-enqueue-time-hook), which is handed a `Metadata` carrying the input |

The same applies to values passed through `Activate(input, otherInputs)`. A value a later junction
reads is produced by an earlier junction.

## If the upgrade is blocked on it

The input-reading fault compiles cleanly and only surfaces at startup. While chains are being
moved over, the check can be turned off:

```csharp
.AddMediator(mediator => mediator.SkipChainVerification())
```

It silences every other chain fault too, so it is a stopgap to remove once the trains are moved,
not a setting to leave on. See
[Trains & Junctions](/docs/core/trains-and-junctions#the-host-checks-every-chain-before-it-serves-traffic)
for the full list of what the check refuses.
