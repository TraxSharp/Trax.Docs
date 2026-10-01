---
layout: default
title: Decision Recording and Models
description: How Trax.Effect records each decision against its run, replays them when a run is re-queued, and answers questions with Nimble or another typed model.
parent: Effect
nav_order: 8
---

# Decision Recording and Models

A train's [decisions](/docs/core/decisions) run without Trax.Effect; Trax.Core needs only an
`IDecider`. Trax.Effect adds three things around them: an adapter for typed decision models, a
record of every decision against the run that made it, and re-queued runs that take the tracks
the original took.

```csharp
services.AddTrax(trax => trax.AddEffects(effects => effects
    .UsePostgres(connectionString)
    .AddDecisionRecording()
    .AddNimbleDecider()));   // Nimble on a local Ollama
```

## Nimble

[Nimble](https://docs.bespokelabs.ai/nimble/overview) is Bespoke Labs' typed decision model, and
the one Trax builds around: its 9B weights are open (published under Apache 2.0, a fine-tune of
Qwen3.5-9B), so a train's decisions can run on hardware you own, with no data leaving it and no
per-request price. Bespoke also hosts it, for when a GPU of your own is not worth running.

| | Local | Hosted |
|---|---|---|
| Register | `AddNimbleDecider()` | `AddNimbleDecider(o => o.ApiKey = key)` |
| Endpoint | `http://localhost:11434/v1/systemone` (Ollama) | `https://api.bespokelabs.ai/v1/systemone` |
| Model | `nimble:9b` | `nimble-v3` |
| Requests in flight | no limit | 8, Bespoke's limit per organisation |
| Before the first run | `ollama pull nimble:9b` | a Bespoke Labs API key |

Both take up to 64 questions a request and 255 options or levels a question; a `Decide` asking
more is refused before it is sent. An attempt may take 30 seconds by default, because a local
Nimble without a large GPU answers in seconds rather than the tens of milliseconds the hosted one
does. On Bespoke's benchmarks it is slightly less accurate than Jev (90.1% against 93.2% on their
held-out set); put it in front of a larger model with a
[cascade](#putting-the-model-in-front-of-a-larger-one) when that matters, and check its
confidence bars against your own labelled cases either way.

Point `Endpoint` at an Ollama on another machine, over HTTPS, to share one GPU between hosts.

## Other typed decision models

`Trax.Effect.Decisions.SystemOne` answers a train's questions through the **System One** request
format, which Nimble shares with Jev (which introduced it), d1, Laya, Kev, OpenDecider and other
models: one `POST` carrying the model, the state, and the questions keyed by name, answered with
one typed answer per question and the model's version. `AddNimbleDecider` is this adapter with
Nimble's endpoints and limits filled in; `AddSystemOneDecider` is the same adapter for any other
model, so moving between them is a change of endpoint and model name:

```csharp
effects.AddSystemOneDecider(o =>
{
    o.Endpoint = new Uri("https://api.typesafe.ai/v1/systemone");
    o.Model = "jev-1.13.0";
    o.ApiKey = configuration["Jev:ApiKey"];
});
```

| Trax question | Sent as | Criteria sent | Read back |
|---|---|---|---|
| Choice | `choice` | each option's name and description | `choice`, `confidence`, `probabilities` |
| Score | `score` | each level's description, lowest first | `score`, `confidence`, `probabilities` by level index |
| Yes/no | `noul` | what yes and no mean (default `Yes` and `No`) | `noul`, the probability of yes |

The state is sent as a string when it is one, and otherwise as JSON with camel-cased names, so a
question's instructions can name a field. Send a state that holds only what the questions need:
these models answer less accurately with detail that does not bear on the question.

[AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider) and
[AddSystemOneDecider](/docs/sdk-reference/configuration/add-system-one-decider) refuse, at
startup, settings that would make decisions hard to trust or leak data:

- **The model must be pinned to a version** (`nimble-v3` or `nimble:9b`, not `nimble-latest`,
  `nimble:latest` or a bare `nimble`). An Ollama tag such as `nimble:9b` moves when the model is
  pulled again, so pull deliberately and re-check the bars after. A confidence bar
  tuned against one version does not carry over to the next. `AllowFloatingModel` turns this off.
- **The endpoint must be HTTPS** unless it is a loopback address, where a self-hosted model
  usually runs. The request carries the train's state and the API key.

A throttled, unavailable or slow model is retried, honouring `Retry-After`; when retries run out
the failure is classified `Transient`. A request the model
refuses (bad criteria, a bad key, no credit left) is classified `Permanent` and not retried. The
failure carries the model's own message and, from Nimble, its request id. Cancelling the run
cancels the request. An answer the adapter cannot read is left out, and the train fails on the
unanswered question rather than acting on a guess.

### Putting the model in front of a larger one

The adapter registers itself as the `IDecider`, and also as `SystemOneDecider`. To escalate what
it is unsure of, register a `CascadingDecider` as the `IDecider` after it:

```csharp
services.AddSingleton<IDecider>(sp => new CascadingDecider(
    sp.GetRequiredService<SystemOneDecider>(),
    sp.GetRequiredService<LargeModelDecider>(),
    escalateBelow: 0.8));
```

See [Escalating what the fast decider is unsure of](/docs/core/decisions#escalating-what-the-fast-decider-is-unsure-of).

## Recording decisions

`AddDecisionRecording()` records, for every question a run asks, the question as asked, the answer
it acted on, the model and decider that gave it, any shadow's answer and whether it agreed, and the
track it took. Each decision is logged as it is made, and written to `trax.decision` against the
run's metadata when the run finishes, however it finishes, so a run that fails after deciding
still shows what it decided.

| Column | Holds |
|---|---|
| `metadata_id` | The run. Rows are deleted with it. |
| `question_key` | The enum or marker type's name |
| `occurrence` | Which asking of the question this was in the run, from 0 |
| `kind` | `choice`, `score` or `yes_no` |
| `question` | The question's instructions and criteria (jsonb) |
| `answer` | The answer acted on (jsonb) |
| `model` | The model and version that answered, or null for a decider that is not a model |
| `decider` | The decider's type, or null for a replayed answer |
| `replayed` | Whether the answer came from an earlier run |
| `shadows` | Each shadow's answer, whether it agreed, and why it gave none (jsonb) |
| `track` | The track a routing step took on this decision |
| `fallback_reason` | Why the decision was not followed, when it was not |
| `decided_at` | When it was answered |

It needs a data provider, and is a compile error before one. The table ships in the core
migration set (Postgres `054`, Sqlite `019`) and is read through `IDataContext.RecordedDecisions`.

```sql
-- How often each support track was taken this week, and how often it was overruled.
SELECT d.track, count(*) AS runs, count(d.fallback_reason) AS overruled
FROM trax.decision d
JOIN trax.metadata m ON m.id = d.metadata_id
WHERE d.question_key = 'TicketTrack' AND m.start_time > now() - interval '7 days'
GROUP BY d.track;
```

That is the data a confidence bar should be tuned from: label a few hundred recorded decisions
with what should have happened, and pick the bar where the cost of a wrong track meets the cost
of sending work to the fallback.

## Re-queued runs replay their decisions

Re-queueing an execution, with the dashboard's **Re-queue** button or the
[`requeueExecution`](/docs/sdk-reference/graphql-api/mutations#requeueexecution) mutation, queues a
run that replays the original's recorded decisions. The new run's metadata carries
`ReplayDecisionsOf`, the original's id; each question it asks is answered from what the original
recorded for the same question and asking, without calling a decider, and recorded with
`replayed` set. A re-queue repeats a run, usually because something after a decision failed, and
asking a model again could take a different track. `Trax.Docs/adr/0041` records why.

A question the original never asked (the chain changed since) is asked afresh, and so is every
question when the original recorded nothing, or was deleted. A replay whose recorded decisions
cannot be read fails the run instead of asking afresh.

To queue a replay yourself, pass the original's metadata id:

```csharp
await operations.QueueTrainAsync(
    new QueueTrainInput(trainName, inputJson) { ReplayDecisionsOf = originalMetadataId },
    ct);
```

## SDK Reference

> [AddDecisionRecording](/docs/sdk-reference/configuration/add-decision-recording) | [AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider) | [AddSystemOneDecider](/docs/sdk-reference/configuration/add-system-one-decider) | [IOperationsService](/docs/sdk-reference/scheduler-api/i-operations-service) | [TrainExecution](/docs/sdk-reference/mediator-api/train-execution)
