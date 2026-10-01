---
layout: default
title: AddDecisionRecording
description: Reference for AddDecisionRecording, which writes every decision a run makes to trax.decision and makes a re-queued run replay the original's decisions.
parent: Configuration
grand_parent: SDK Reference
nav_order: 21
---

# AddDecisionRecording

Records every [decision](/docs/core/decisions) a train makes against the run that made it, in
`trax.decision`, and makes a re-queued run replay the decisions of the run it repeats. Each
decision is also logged as it is made. See
[Decision Recording and Models](/docs/effect/decisions).

## Signature

```csharp
public static TraxEffectBuilderWithData AddDecisionRecording(
    this TraxEffectBuilderWithData configurationBuilder
)
```

Defined on `TraxEffectBuilderWithData`, so it comes after a data provider. Called before one, it
is a compile error that says so.

## Returns

`TraxEffectBuilderWithData`, for continued chaining.

## Example

```csharp
services.AddTrax(trax => trax
    .AddEffects(effects => effects
        .UsePostgres(connectionString)
        .AddDecisionRecording()
    )
);
```

## What it registers

| Service | Lifetime | Role |
|---|---|---|
| `DecisionJournal` | Singleton | Holds each run's decisions until it finishes, and serves a replaying run its original's answers |
| `IDecisionObserver` | Singleton | The journal, which Trax.Core tells about every decision and routing |
| `IDecisionReplay` | Singleton | The journal, which Trax.Core asks before it asks a decider |
| `DecisionRecordingHook` | Lifecycle hook | Loads the decisions a run replays when it starts, and writes a run's decisions when it finishes, however it finishes |

## Remarks

- The hook is not toggleable: the journal holds a run's decisions until the hook writes them.
- A run that is not persisted (no metadata row) has its decisions logged and not written.
- Writing the decisions never fails the run; a failed write is logged with the run's id.
- A run whose metadata names an earlier run in `ReplayDecisionsOf` replays it. If the earlier
  run's decisions cannot be read, the run's first decision fails instead of asking afresh.
- A host with no logging registered still records.

## Package

```
dotnet add package Trax.Effect.Data
```
