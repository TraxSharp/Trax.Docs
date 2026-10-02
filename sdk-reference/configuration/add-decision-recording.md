---
layout: default
title: AddDecisionRecording
description: Reference for AddDecisionRecording, which writes every decision a run makes to trax.decision and makes a re-queued or retried run replay them.
parent: Configuration
grand_parent: SDK Reference
nav_order: 21
---

# AddDecisionRecording

Records every [decision](/docs/core/decisions) a train makes against the run that made it, in
`trax.decision`, as each decision is made, and makes a re-queued or retried run replay the
decisions of the run it repeats, into the same state and only while they are fresh. Each decision is also logged. See
[Decision Recording and Models](/docs/effect/decisions).

## Signature

```csharp
public static TraxEffectBuilderWithData AddDecisionRecording(
    this TraxEffectBuilderWithData configurationBuilder
)

public static TraxEffectBuilderWithData AddDecisionRecording(
    this TraxEffectBuilderWithData configurationBuilder,
    Action<DecisionRecordingOptions> configure
)
```

| Parameter | Type | Description |
|---|---|---|
| `configure` | `Action<DecisionRecordingOptions>` | Sets how long a recorded answer is replayed for. Called again, it changes the same options. |

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

## DecisionRecordingOptions

```csharp
// Trax.Effect.Data.Decisions
public sealed class DecisionRecordingOptions
{
    public static readonly TimeSpan DefaultMaxReplayAge;   // 24 hours
    public TimeSpan MaxReplayAge { get; }
    public DecisionRecordingOptions ReplayAnswersFor(TimeSpan maxAge);
}
```

| Member | Default | Description |
|---|---|---|
| `ReplayAnswersFor(TimeSpan)` | 24 hours | Replays a recorded answer into a repeated run only while it is younger than this, counted from when a decider gave it, not from a later run that replayed it. An older answer is asked afresh and the reason logged at `Information`. Throws `ArgumentOutOfRangeException` under one second. |
| `MaxReplayAge` | 24 hours | The bound in effect |

```csharp
effects.UsePostgres(connectionString)
    .AddDecisionRecording(o => o.ReplayAnswersFor(TimeSpan.FromHours(6)))
```

## What it registers

| Service | Lifetime | Role |
|---|---|---|
| `DecisionJournal` | Singleton | Writes each decision as it is made, and serves a replaying run its original's answers |
| `IDecisionObserver` | Singleton | A composite that tells the journal, and every other observer, about every decision and routing. The journal is `Required` and told first, so a failed write fails the step. See [Other decision observers](/docs/effect/decisions#other-decision-observers). |
| `DecisionRecordingOptions` | Singleton | The options `configure` set |
| `IDecisionReplay` | Singleton | The journal, which Trax.Core asks before it asks a decider |

Calling `AddDecisionRecording()` more than once registers these once.

## Remarks

- Each decision is written before the train acts on it, through a short-lived data context from
  `IDataContextProviderFactory` rather than the run's own, so it never flushes what the run's
  junctions have tracked. Each track a routing step takes is added, in order, to the `routes` of
  the latest row for that question when it is taken. A run killed mid-way keeps every decision it
  acted on.
- Each decision is stored with the fingerprint Trax.Core reports for it, and a replay hands the
  fingerprint back with the answer, so an answer given to a different asking of the question is
  not replayed.
- Each decision is stored with `DecisionMade.StateHash` in `state_hash`, and a replay hands it back
  as `RecordedAnswer.StateHash`, so Trax.Core replays the answer only into a state that hashes the
  same. A row with no hash (the state could not be written as JSON, or the row predates the column)
  is never replayed. See
  [Only into the same state, and only while fresh](/docs/effect/decisions#only-into-the-same-state-and-only-while-fresh).
- An answer older than `ReplayAnswersFor` is left out of the replay, so the question is asked
  afresh. The age counts from the row a decider wrote, following replayed rows back to it; a
  replayed row whose answering run is no longer in the chain is asked afresh.
- An `IDecisionObserver` registered after `AddTrax` would replace the composite, so the journal
  would never be told. The host then refuses to start, and every run that would record its
  decisions refuses too, with an `InvalidOperationException` naming the observer. Register your own
  observer before `AddTrax`.
- A decision that cannot be written fails its step before any track is taken, classified
  `Transient`, or `Permanent` when the store refuses the value. A live answer of a type the journal
  has no stored form for fails its step, `Permanent`, because it could never be read back.
- A shadow's answer cannot cost the live record: a number JSON cannot hold is written as the
  string `"NaN"`, `"Infinity"` or `"-Infinity"`, and an answer of a type with no stored form is
  recorded with the reason in the shadow's error.
- A decider's missing or unfit live answer, which fails its step, is recorded too, with the answer
  (null when there was none), the decider, the model and the reason in `refused`. A refused row is
  never replayed, so a requeue of that run asks the question afresh.
- Each run's metadata row is marked `DecisionsRecorded` on its first write, before any junction.
- A decision, refusal, routing or replay lookup that arrives without the run on its async flow,
  because code in the run suppressed `ExecutionContext` flow, is matched to the run by external id
  among the runs of that train in progress on this host. Exactly one match is recorded against;
  otherwise it is logged only.
- A run that is not persisted (no metadata row) has its decisions logged and not written. A
  decision the run's own train reports under an external id other than the run's, because the
  train changed its `ExternalId` while running, fails its step, `Permanent`.
- A run whose metadata names an earlier run in `ReplayDecisionsOf` loads, before its first
  junction, the answers of that run and, for questions it never reached, of the runs it replayed
  in turn, nearest first, up to 32 runs back. It fails there, classified `Permanent`, rather than
  asking afresh, when a run in that chain does not exist, is a run of another train, or ran without
  recording its decisions, when the chain loops or goes back further, or when a recorded answer
  cannot be read; and classified `Transient` when the database fails. A question no run in the
  chain reached is asked afresh. See
  [A requeue of a requeue](/docs/effect/decisions#a-requeue-of-a-requeue).
- On a host without `AddDecisionRecording()`, a run that names a run to replay fails before its
  first junction, classified `Permanent`, because it has nothing to replay from.
- Runs that share an external id, as a retried dispatch's rows can, each write under their own
  row and replay their own answers.
- A host with no logging registered still records.
- A manifest's retry and a requeue of its dead letter name a run to replay only under the
  scheduler's checks. See
  [Retries replay decisions](/docs/scheduler/dead-letters-and-cleanup#retries-replay-decisions).

## HasDecisionsToReplay

```csharp
// Trax.Effect.Data.Decisions
public static Task<bool> HasDecisionsToReplay(
    this IDataContext context,
    long metadataId,
    CancellationToken cancellationToken
)
```

Whether a requeue of the run `metadataId` should replay its decisions rather than be queued as an
ordinary run that asks afresh. True when the run recorded a decision it acted on (a refused answer
does not count, since it is never replayed), or was itself queued to
replay another run's: a requeue of a requeue that failed before reaching a question recorded
nothing, but the answers of the run it replayed are still the ones to repeat. False for a run of a
train that never decides, a run on a host that records nothing, and a run that does not exist.
[`IOperationsService.RequeueExecutionAsync`](/docs/sdk-reference/scheduler-api/i-operations-service#requeueexecutionasync)
asks it; a host that requeues runs its own way should too.

## Package

```
dotnet add package Trax.Effect.Data
```
