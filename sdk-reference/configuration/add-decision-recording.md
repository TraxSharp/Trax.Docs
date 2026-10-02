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
`trax.decision`, as each decision is made, and makes a re-queued run replay the decisions of the
run it repeats. Each decision is also logged. See
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
| `DecisionJournal` | Singleton | Writes each decision as it is made, and serves a replaying run its original's answers |
| `IDecisionObserver` | Singleton | The journal, which Trax.Core awaits for every decision and routing. It is `Required`, so a failed write fails the step. |
| `IDecisionReplay` | Singleton | The journal, which Trax.Core asks before it asks a decider |

Calling `AddDecisionRecording()` more than once registers these once.

## Remarks

- Each decision is written before the train acts on it, through a short-lived data context from
  `IDataContextProviderFactory` rather than the run's own, so it never flushes what the run's
  junctions have tracked. The track a routing step takes is written onto the latest row for that
  question when it is taken. A run killed mid-way keeps every decision it acted on.
- A decision that cannot be written fails its step before any track is taken, classified
  `Transient`, or `Permanent` when the store refuses the value.
- A shadow's answer cannot cost the live record: a number JSON cannot hold is written as the
  string `"NaN"`, `"Infinity"` or `"-Infinity"`.
- A run that is not persisted (no metadata row) has its decisions logged and not written.
- A run whose metadata names an earlier run in `ReplayDecisionsOf` loads that run's answers before
  its first junction. If that run does not exist, the run fails there, classified `Permanent`,
  rather than asking afresh. A question the earlier run never reached is asked afresh.
- On a host without `AddDecisionRecording()`, a run that names a run to replay fails before its
  first junction, classified `Permanent`, because it has nothing to replay from.
- Runs that share an external id, as a retried dispatch's rows can, each write under their own
  row and replay their own answers.
- A host with no logging registered still records.

## Package

```
dotnet add package Trax.Effect.Data
```
