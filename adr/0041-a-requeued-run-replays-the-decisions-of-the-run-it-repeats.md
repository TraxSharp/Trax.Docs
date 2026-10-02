---
authors: [Theauxm]
repos: [effect, mediator, scheduler, api, dashboard]
areas: [platform, data-model, graphql]
status: accepted
---

# A requeued run replays the decisions of the run it repeats

Re-queueing an execution, from the dashboard or through `requeueExecution`, queues a run that
replays the decisions the original recorded: it takes the tracks the original took instead of
asking its deciders again. Both surfaces call `IOperationsService.RequeueExecutionAsync`, which is
the only place the link is set: to the run being re-queued, and only when that run recorded
decisions. The link travels from the queued entry to the new run's metadata as
`replay_decisions_of`, and a run with recorded answers for a question uses them in place of a
decider's. An ordinary enqueue (`queueTrain`, `QueueTrainAsync`), a dead-letter retry and a
manifest's scheduled run do not replay.

## Status

**Accepted.** Builds on [0040](./0040-a-chain-declares-every-track-and-a-decider-chooses-one.md).

## Why this is written down

Because the alternative is the obvious one, and it is wrong for the reason people requeue.

A re-queue repeats a run, usually because something downstream of a decision failed. Asking a
model again can get a different answer to the same question, so the repeat would take a
different track and do different work from the run it was meant to repeat, with nothing on
screen to say so. Replaying makes the repeat do what the original decided.

## Considered options

**Ask afresh on every run.** Simplest, and what a run with nothing recorded still does.
Rejected as the default for a re-queue for the reason above.

**Key the replay by input.** Rejected: two runs with the same input are not the same run, and a
train that is run often with one input would replay a decision from an unrelated run.

**Carry the link in the existing `parent_id`.** Rejected: `parent_id` means a nested child run,
and the run views filter on it, so a re-queued run would vanish from the root-level lists.

## Consequences

**The link is a column on `work_queue` and on `metadata`, not a foreign key.** The original may
be deleted before the re-queue runs (cleanup, manifest pruning). A run whose original no longer
exists fails before its first junction, classified permanent, rather than asking afresh.

**Recording is what makes replay possible.** Decisions are written to `trax.decision` by
`AddDecisionRecording`, against the run, as each is made and before the train acts on it, through
a short-lived data context of its own. A run killed mid-way (an out-of-memory kill, a timeout, a
deploy) keeps every decision it acted on, so its re-queue can repeat them. The journal is a
`Required` observer: a decision that cannot be written fails its step, classified transient,
because acting on a decision a later replay cannot see would make the replay ask afresh.

**A replay that cannot be honoured fails the run.** A run that names an original to replay fails
before its first junction, classified permanent, when the host does not record decisions or the
original does not exist, and fails there too when its decisions cannot be loaded, rather than
quietly asking again,
because the run was queued to repeat the original. Two things are still asked afresh: a question
the original never reached, and a recorded answer that no longer fits the question as it is asked
now (an option renamed or no longer offered, a scale with fewer levels). The second is noted in the
stored answer as `replay_refused`.

**An execution service that cannot carry the link is a misconfiguration.** A custom or decorating
`ITrainExecutionService` that predates the `QueueTrainOptions` overload throws
`DecisionReplayNotSupportedException` when asked to queue a replay, and the requeue logs and
throws it rather than reporting it as a refused enqueue or queueing without the link.

**The run is identified by its row.** A run started from a row the scheduler created takes that
row's external id, so the decisions it records and the failure data it carries name the run they
belong to. Before this, such a run kept the random id its train instance was constructed with.

**Changing the queue API without breaking it.** `ITrainExecutionService` gained an overload that
takes `QueueTrainOptions`, with a default implementation that refuses a replay rather than
dropping it. `IOperationsService` gained `RequeueExecutionAsync`, with a default implementation
that throws, so an implementation written before it still compiles. `QueueTrainInput` does not
carry the link: the requeue owns it.

## Exemplars

**Enforced elsewhere:** `DecisionRecordingTests` in Trax.Effect pins that a run records each
decision and the track it took against its metadata, that a decision is written before the run
ends, that a decision that cannot be written fails its step as transient, that a decision not
followed records why, that a run that fails after deciding still records, that a run naming an
earlier one replays its decisions without asking, that a replay of a run that does not exist or on
a host that records no decisions fails the run as permanent, that questions the original never
reached are asked afresh, and that a run started from a row takes the row's external id;
`SqliteDecisionRecordingTests` and `PostgresDecisionRecordingTests` run the same against real
stores. `QueueInputTests` in Trax.Mediator pins that the options overload stores the link and that
an implementation predating it throws `DecisionReplayNotSupportedException`.
`OperationsServiceRequeueTests` in Trax.Scheduler pins that a run that recorded decisions is
re-queued to replay them, that one that recorded none is re-queued as an ordinary enqueue, that a
caller's queue input has no replay link, and that a replay through an execution service without
the overload is a misconfiguration; `JobDispatcherTrainTests` pins the link onto the dispatched
run's metadata. `OperationsQueriesTests` in Trax.Api and `MetadataRequeueRefusalTests` and
`MetadataRequeueTrustedScopeTests` in Trax.Dashboard pin that both re-queue surfaces go through
the shared requeue.

Not covered: a train whose chain changed between the original run and its re-queue replays the
questions the two have in common and asks the new ones afresh; nothing flags that the replayed
run took a path the original chain could not have.

## Changelog

- **2026-10-01**: Decisions are written as each is made rather than when the run finishes; a
  replay that cannot be honoured (no recording on the host, no such run) fails the run instead of
  asking afresh; the link is set only by `IOperationsService.RequeueExecutionAsync`, not by the
  queue input.
- **2026-10-01**: Recorded.
