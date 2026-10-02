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
    .AddNimbleDecider(o => o.Endpoint = new Uri("http://localhost:8000/v1/systemone"))));
```

## Nimble

[Nimble](https://docs.bespokelabs.ai/nimble/overview) is Bespoke Labs' typed decision model, and
the one Trax builds around: its 9B weights are open (published under Apache 2.0, a fine-tune of
Qwen3.5-9B), so a train's decisions can run on hardware you own, with no data leaving it and no
per-request price.

`AddNimbleDecider` talks to a Nimble server you run, started from Nimble's own serving code
(`nimble/serving/server.py` in `bespokelabsai/nimble`), which answers `POST /v1/systemone`. There
is no default endpoint. Nimble's documentation describes no production hosted API, and the public
demo it links to is unauthenticated and runs on one GPU, so it is not somewhere to send a train's
state and Trax does not point at it. A host that registers Nimble without an `Endpoint` fails to
start.

| Setting | Default | Why |
|---|---|---|
| `Endpoint` | none, required | The full URL of `POST /v1/systemone` on your server |
| `Model` | `bespokelabs/Bespoke-Nimble-9B` | The checkpoint id the server answers to. `nimble-latest` is refused. |
| `ApiKey` | none | Sent as a bearer token only when set. The server checks one only when it is started with `OPENJEV_API_KEY`. |
| `MaxConcurrentRequests` | 4 | One server container runs four evaluations at once and turns a fifth away with a 529. Raise it when the server scales to more containers. |
| `MaxOptions` | 26 | The most options or levels the server accepts on one question |
| Questions per request | 64 | The server's limit; a `Decide` asking more is refused before it is sent |
| `AttemptTimeout` | 30 seconds | A self-hosted server can be slow on its first request after loading |

The request cannot pin a model revision: the server serves whichever revision it was deployed
with. Pin the revision where you deploy the server, and re-check your confidence bars when you
change it. Token and body limits (8,192 prompt tokens a question, a 2 MiB body) cannot be checked
before sending, so a request over them is answered 413 or 422 and fails as `Permanent`.

On Bespoke's benchmarks Nimble is slightly less accurate than Jev (90.1% against 93.2% on their
held-out set); put it in front of a larger model with a
[cascade](#putting-the-model-in-front-of-a-larger-one) when that matters, and check its
confidence bars against your own labelled cases either way.

## Other typed decision models

`Trax.Effect.Decisions.SystemOne` answers a train's questions through the **System One** request
format, which Nimble shares with Jev (which introduced it), d1, Laya, Kev, OpenDecider and other
models: one `POST` carrying the model, the state, and the questions, each with its
[question key](/docs/core/decisions#question-keys) as its id, answered with one typed answer per
question and the model's version. `AddNimbleDecider` is this adapter with Nimble's model name and
limits filled in; `AddSystemOneDecider` is the same adapter for any other model, so moving between
them is a change of endpoint and model name:

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
| Choice | `choice` | each option's name and description | `choice`, `confidence` (or, without one, the chosen option's probability), `probabilities` |
| Score | `score` | each level's description, lowest first | `score`, `confidence` (or, without one, the nearest level's probability), `probabilities` keyed by every level index from 0 |
| Yes/no | `noul` | what yes and no mean (default `Yes` and `No`) | `noul`, the probability of yes |

The state is sent as a string when it is one, and otherwise as JSON with camel-cased names, so a
question's instructions can name a field. Send a state that holds only what the questions need:
these models answer less accurately with detail that does not bear on the question.

[AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider) and
[AddSystemOneDecider](/docs/sdk-reference/configuration/add-system-one-decider) refuse, at
startup, settings that would make decisions hard to trust or leak data:

- **There must be an endpoint**, an `http` or `https` URL. Nimble has no default to fall back on.
- **The model must be pinned to a version** (`jev-1.13.0`, not `jev-latest` or a bare `jev`;
  Nimble's `bespokelabs/Bespoke-Nimble-9B`, not `nimble-latest`). A confidence bar tuned against
  one version does not carry over to the next. `AllowFloatingModel` turns this off for
  `AddSystemOneDecider`.
- **The endpoint must be HTTPS** unless it is a loopback address, where a self-hosted model
  usually runs. The request carries the train's state and the API key.

The API key is optional; a blank one, such as an unset configuration value, sends no
`Authorization` header.

A throttled, unavailable or slow model, or a `200` whose body is not a System One response, is
retried with a doubling, jittered wait that stops growing at `MaxRetryDelay` (30 seconds by
default). A `Retry-After` is honoured up to `MaxRetryDelay`; when the model asks for longer, or the
retries run out, the failure is classified `Transient`. A request the model refuses (bad criteria,
a bad key, a `501` or `505`) is classified `Permanent` and not retried, and so is one that cannot be
sent as it is: a state that cannot be serialized, an unsupported question type, a duplicate option
or question key. The failure carries the model's own message and, from Nimble, its request id.
Cancelling the run cancels the request. An answer the adapter cannot read is left out, and the
train fails on the unanswered question rather than acting on a guess.

### Putting the model in front of a larger one

Without a name, each method registers the decider every train asks, as both `IDecider` and
`SystemOneDecider`, and may be called once; a second unnamed registration fails at startup instead
of replacing the first. With a name, it registers a keyed `SystemOneDecider` and nothing else, so
several models can sit side by side. Compose them yourself, and register the result as the
`IDecider`:

```csharp
services.AddTrax(trax => trax.AddEffects(effects => effects
    .AddNimbleDecider("nimble", o => o.Endpoint = new Uri("http://localhost:8000/v1/systemone"))
    .AddSystemOneDecider("jev", o =>
    {
        o.Endpoint = new Uri("https://api.typesafe.ai/v1/systemone");
        o.Model = "jev-1.13.0";
        o.ApiKey = configuration["Jev:ApiKey"];
    })));

services.AddSingleton<IDecider>(sp => new CascadingDecider(
    sp.GetRequiredKeyedService<SystemOneDecider>("nimble"),
    sp.GetRequiredKeyedService<SystemOneDecider>("jev"),
    escalateBelow: 0.8));
```

A keyed decider is not something `DecidedBy<TDecider>()` can name, because that resolves by type.
To give one step a different decider, register a type of your own for it.

See [Escalating what the fast decider is unsure of](/docs/core/decisions#escalating-what-the-fast-decider-is-unsure-of).

## Recording decisions

`AddDecisionRecording()` records, for every question a run asks, the question as asked, the answer
it acted on, the model and decider that gave it, any shadow's answer and whether it agreed, and the
track it took. Each decision is logged and written to `trax.decision` against the run's metadata
as it is made, before the train acts on it, through a short-lived data context of its own rather
than the run's. A run that fails, times out or is killed after deciding still shows what it
decided, and the track a routing step takes is written onto that question's row when it is taken.

Recording is required, not best effort. A decision that cannot be written fails its step before
any track is taken, [classified](/docs/core/trains-and-junctions#classifying-failures) `Transient`
(or `Permanent` when the store refuses the value), because a re-queue of that run would otherwise
have nothing to replay. A shadow's answer can never cost the live record: a number JSON cannot
hold is written as the string `"NaN"`, `"Infinity"` or `"-Infinity"`. Calling
`AddDecisionRecording()` more than once registers it once.

| Column | Holds |
|---|---|
| `metadata_id` | The run. Rows are deleted with it. |
| `question_key` | The [question key](/docs/core/decisions#question-keys): the enum or marker type's full name |
| `occurrence` | Which asking of the question this was in the run, from 0 |
| `kind` | `choice`, `score` or `yes_no` |
| `question` | The question's instructions and criteria (jsonb) |
| `answer` | The answer acted on (jsonb), with `replay_refused` when an earlier run's answer no longer fitted and the decider was asked afresh |
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
-- question_key is the type's full name: TicketTrack, declared in the Support namespace.
SELECT d.track, count(*) AS runs, count(d.fallback_reason) AS overruled
FROM trax.decision d
JOIN trax.metadata m ON m.id = d.metadata_id
WHERE d.question_key = 'Support.TicketTrack' AND m.start_time > now() - interval '7 days'
GROUP BY d.track;
```

That is the data a confidence bar should be tuned from: label a few hundred recorded decisions
with what should have happened, and pick the bar where the cost of a wrong track meets the cost
of sending work to the fallback.

## Re-queued runs replay their decisions

Re-queueing an execution, with the dashboard's **Re-queue** button or the
[`requeueExecution`](/docs/sdk-reference/graphql-api/mutations#requeueexecution) mutation, queues a
run that replays the original's recorded decisions. Both go through
[`IOperationsService.RequeueExecutionAsync`](/docs/sdk-reference/scheduler-api/i-operations-service#requeueexecutionasync),
which sets the new run's `ReplayDecisionsOf` to the original's id, and only when the original
recorded decisions; a run of a train that never decides is re-queued as an ordinary enqueue. Each
question the new run asks is answered from what the original recorded for the same question and
asking, without calling a decider or its shadows, and recorded with `replayed` set. A re-queue
repeats a run, usually because something after a decision failed, and asking a model again could
take a different track. `Trax.Docs/adr/0041` records why.

The requeue is the only way to set the link. Queueing a train through `queueTrain` or
`QueueTrainAsync` never replays, and neither does a dead-letter retry or a manifest's scheduled run.

| The replay meets | What happens |
|---|---|
| A question the original never reached (it failed earlier, or the chain changed since) | Asked afresh |
| A recorded answer that no longer fits the question (an option renamed or no longer offered, a scale with fewer levels, another kind of question) | Asked afresh, with the reason stored as `replay_refused` |
| A host that does not call `AddDecisionRecording()` | The run fails before its first junction, `Permanent` |
| An original run that no longer exists | The run fails before its first junction, `Permanent` |
| Recorded decisions that cannot be read | The run fails rather than asking afresh |

A run that cannot honour its replay fails instead of asking afresh, because it was queued to
repeat the original.

## SDK Reference

> [AddDecisionRecording](/docs/sdk-reference/configuration/add-decision-recording) | [AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider) | [AddSystemOneDecider](/docs/sdk-reference/configuration/add-system-one-decider) | [IOperationsService](/docs/sdk-reference/scheduler-api/i-operations-service) | [TrainExecution](/docs/sdk-reference/mediator-api/train-execution)
