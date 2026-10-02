---
layout: default
title: Junction Events
description: "AddJunctionEvents: each step of a run published live and stored in trax.junction_run, what a step carries, SignalR, RabbitMQ, GraphQL and the dashboard."
parent: Effect
nav_order: 9
---

# Junction Events

A train's lifecycle events say that a run started, completed or failed. Junction events say what
happened in between: each junction starting and ending, each question a routing step asked and the
answer the run acted on, and each track it took. They are published live and stored in
`trax.junction_run`, so a running run can be followed step by step and a finished one read back as
a timeline.

They are off unless the host calls `AddJunctionEvents()`:

```csharp
services.AddTrax(trax => trax.AddEffects(effects => effects
    .UsePostgres(connectionString)
    .AddJunctionEvents()));
```

It comes after a data provider, and is a compile error before one. Calling it more than once
registers it once. `Trax.Effect/docs/adr/0019` records why each of the rules below is the way it is.

## What a step is

| Kind | Step | Event types |
|---|---|---|
| `Junction` | An `EffectJunction` that ran | `JunctionStarted`, then one of `JunctionCompleted`, `JunctionFailed`, `JunctionCancelled` |
| `Choice` | A question answered by choosing an option (`Decide`, `Switch`) | `Decided`, or `DecisionRefused` when the run would not act on the answer |
| `Score` | A question answered by a level (`Scale`) | `Decided` or `DecisionRefused` |
| `YesNo` | A question answered by the probability of yes (`Gate`) | `Decided` or `DecisionRefused` |
| `Route` | The track a routing step sent the run down | `Routed` |

Each step has a `Position` from 0, in the order the run reached it. A junction's start and end
share one position, so ordering by position gives the run's timeline. Only `EffectJunction`s are
steps: a plain `Junction` in a service train runs no junction effects and reports nothing, and a
junction skipped because an earlier one failed is not a step.

Trax.Core reports a decision without naming the routing step that asked it, so a question is told
apart by its kind rather than by `Switch` or `Gate`.

## What a step carries, and never carries

Each event is a `TrainLifecycleEventMessage` with a junction event type and its step in `Junction`,
a `JunctionEventPayload`:

| Field | Holds |
|---|---|
| `Position`, `Kind`, `Name` | Where the step falls, what it is, and the junction's class name without its namespace, or a question's key |
| `State` | `InProgress`, `Completed`, `Failed` or `Cancelled` |
| `StartedAt`, `EndedAt`, `DurationMs` | UTC times, and the junction's duration once it ends |
| `FailureClass`, `FailureException` | How a failed junction's failure is [classified](/docs/core/trains-and-junctions#classifying-failures), and its exception's type name |
| `QuestionKey`, `Answer`, `Confidence` | For a question or track: its [key](/docs/core/decisions#question-keys), the option, score or probability the run acted on, and the decider's confidence |
| `Replayed` | True when the answer was [replayed](/docs/effect/decisions#re-queued-and-retried-runs-replay-their-decisions) from an earlier run |
| `Decider` | The decider's type name. Live events only. |
| `AnswerWithheld` | True when the question is about a `[TraxSensitive]` type |
| `Attempt` | Which attempt of its manifest the run is, or null for a run with no manifest |

A step never carries a junction's input or output, the train's input or output, a failure's
message, or the state, instructions or criteria of a question. A failed junction is described by its
exception's type and its failure class only.

### Withholding an answer

Mark the enum or marker type a routing step asks about with
[`[TraxSensitive]`](/docs/sdk-reference/attributes/trax-sensitive#on-a-question-type) and its
answers are withheld from every junction event and every `trax.junction_run` row: the step records
that the question was asked and answered, with `AnswerWithheld` set, but not the option, score,
probability, confidence or the track taken on it.

```csharp
[TraxSensitive]
[Asks("Should this account be flagged for fraud review?")]
public sealed class FlagForFraud;
```

The mark is honoured wherever the question's key is built from the marked type: a closed form of a
marked generic type, a type nested in one, a type that takes one as a type argument, and a type
that inherits the mark. A key that merely shares a name with a marked type is withheld too.

Withholding the answer does not hide the path: the junctions a track runs are named in their own
steps. A host whose track choice must stay private should not turn junction events on.
`trax.decision` keeps the full answer either way, because a requeue replays it from there.

## Where steps go

Each step goes to the host's `IJunctionEventHandler`s and, when [`UseBroadcaster`](/docs/sdk-reference/configuration/use-broadcaster)
has a transport, to the junction event handlers of other hosts. A junction event never reaches an
`ITrainEventHandler`, so a handler written for train events is never handed an event type it does
not know.

```csharp
public sealed class StepLogger(ILogger<StepLogger> logger) : IJunctionEventHandler
{
    public Task HandleAsync(TrainLifecycleEventMessage message, CancellationToken ct)
    {
        var step = message.Junction!;
        logger.LogInformation("{Train} step {Position} {Name}: {State}",
            message.TrainName, step.Position, step.Name, step.State);
        return Task.CompletedTask;
    }
}

services.AddScoped<IJunctionEventHandler, StepLogger>();
```

On the host that runs the train, a handler is called on the run's path, from the run's scope, right
after the step is published, so the next junction waits for it: return quickly and queue anything
slow. On other hosts it is called by the receiver, from a fresh scope per message. Whatever a
handler throws is logged and never reaches the run or the other handlers.

Publishing never fails a run and never changes a junction's result. A failure to store, broadcast
or hand out a step is logged and swallowed.

## Stored steps

A background writer stores each step in `trax.junction_run`, one row per position, so the run never
waits on the database. It queues up to 4,096 steps and writes them in order and in batches through
a data context of its own. When the queue is full a step is dropped, counted and logged, rather than
holding up the run. Stopping the host drains what is queued.

Read a run's steps through `IDataContext.JunctionRuns` and `ForRun`, which orders them by position:

```csharp
var steps = await dataContext.JunctionRuns.AsNoTracking().ForRun(metadataId).ToListAsync(ct);
```

The rows can trail the live events by moments. A client following a running run subscribes first,
then reads the stored steps, and keeps for each position whichever is further along. A dropped step
is missing, never wrong.

**Retention.** A row is deleted with its run's metadata row, by the foreign key's cascade, so
[metadata cleanup](/docs/scheduler/admin-trains/metadata-cleanup), manifest pruning and any other
delete of metadata remove a run's steps with it.

The table ships in the core migration set: Postgres `055` and `057`, Sqlite `020` and `022`.

### Attempt

A run of a manifest carries which attempt it is: 1 plus the manifest's failed runs since its last
completed or cancelled one, skipping dispatch attempts the scheduler requeued. It is read once, when
the run begins, from at most the manifest's 1,000 most recent runs through an index on
`(manifest_id, id)` (Postgres `058`, Sqlite `023`), and the run waits on it for at most a second. A
run with no manifest carries none, and a read that fails or times out leaves it out and the run
alone.

## SignalR

The [SignalR sink](/docs/sdk-reference/configuration/use-signalr-hub#junction-events) sends
junction events to clients only when told to:

```csharp
effects.UseBroadcaster(b => b.UseSignalRHub(opts => opts.WithJunctionEvents()))
```

| Option | Sends |
|---|---|
| none (default) | No junction events |
| `WithJunctionEvents()` | Each step as a `TraxJunctionClientEvent`, through the `"JunctionEvent"` client method. A question's key and whether it was replayed, but not its answer or confidence. |
| `WithJunctionAnswers()` | The same, with each question's answer and confidence. Answers to `[TraxSensitive]` questions stay withheld. |
| `WithJunctionProjection<T>(...)` | Each step in a shape of your own, as `WithProjection` does for train events |

Every client the hub admits receives every train's events, so the default payload leaves answers
out. A junction event passes the same `OnlyForTrains` filter as its train's events, and an
`OnlyForEvents` list must name the junction event types to let them through. When the sink's queue
is full it gives up junction events before train events.

## RabbitMQ

With [`UseRabbitMq`](/docs/sdk-reference/configuration/use-broadcaster#rabbitmq), junction events
are published to a fanout exchange of their own, `<ExchangeName>.junctions` by default
(`trax.lifecycle.junctions`), set with `JunctionExchangeName`. It must differ from `ExchangeName`,
or `UseRabbitMq` throws. A receiver of this version binds its queue to both exchanges; a receiver
from before junction events binds only the train exchange, so it never receives one.

That makes the rollout order free. Upgrade a fleet's hosts in any order: an old receiver never sees
a step, an old publisher never sends one, and a new receiver gets steps from new publishers as soon
as they start. Because a run's steps and its train events travel through different exchanges, a
subscriber can see a run's `Completed` before its last step arrives; order by position and
timestamps rather than by arrival. The publisher's outgoing queue gives up junction events first
when it is full, so a busy run's steps never cost another run its outcome.

## GraphQL

Trax.Api forwards junction events to one subscription and reads the stored steps through one query:

| Field | Returns | Who may use it |
|---|---|---|
| [`onJunctionEvent(metadataId: Long!)`](/docs/sdk-reference/graphql-api/subscriptions#onjunctionevent) | The steps of one run, live | Exactly who would receive that run's train events: every train for the operations view, `[TraxBroadcast]` trains whose posture admits the caller for everyone else |
| [`operations.junctionRuns(metadataId, afterPosition, take)`](/docs/sdk-reference/graphql-api/queries#junctionruns) | The stored steps of one run, in order | The operations gate, as `operations.execution` |

The run is a required argument: there is no feed of every run's steps. Outside the operations view
a step loses host detail, as a train event does: the decider's type name, and a failure's exception
type unless it is a `TrainException`. It also carries no `answer` or `confidence` unless the host
calls `AllowJunctionAnswersForBroadcastSubscribers()` on `AddTraxGraphQL`, the same default the
SignalR payload has. The operations view always sees answers, and a `[TraxSensitive]` answer is
withheld from everyone.

## Dashboard

The dashboard's run page draws a **Junction Timeline** from the stored steps: one row per step,
with a bar placed against the run's start and coloured by state, a running step extending to now on
each refresh, a question's key, answer, confidence and whether it was replayed, a withheld answer
shown as "withheld", and a failed step's failure class and exception type. The run's attempt is
shown when its rows carry one. It reads at most the first 500 steps and says so when the run
recorded more. A run with no stored steps shows a hint naming `AddJunctionEvents()`.
See [Dashboard: Metadata Detail Page](/docs/dashboard#metadata-detail-page).

## SDK Reference

> [AddJunctionEvents](/docs/sdk-reference/configuration/add-junction-events) | [UseSignalRHub](/docs/sdk-reference/configuration/use-signalr-hub) | [UseBroadcaster](/docs/sdk-reference/configuration/use-broadcaster) | [TraxSensitive](/docs/sdk-reference/attributes/trax-sensitive) | [Subscriptions](/docs/sdk-reference/graphql-api/subscriptions) | [IDataContext](/docs/sdk-reference/configuration/i-data-context)
