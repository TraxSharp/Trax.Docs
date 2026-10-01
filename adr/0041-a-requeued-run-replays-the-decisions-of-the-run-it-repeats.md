---
authors: [Theauxm]
repos: [effect, mediator, scheduler, api, dashboard]
areas: [platform, data-model, graphql]
status: accepted
---

# A requeued run replays the decisions of the run it repeats

Re-queueing an execution, from the dashboard or through `requeueExecution`, queues a run that
replays the decisions the original recorded: it takes the tracks the original took instead of
asking its deciders again. The link travels from the queued entry to the new run's metadata as
`replay_decisions_of`, and a run with recorded answers for a question uses them in place of a
decider's.

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
be deleted before the re-queue runs (cleanup, manifest pruning); a replay that finds nothing
recorded asks afresh.

**Recording is what makes replay possible.** Decisions are written to `trax.decision` by
`AddDecisionRecording`, against the run, when it finishes, however it finishes. A host without
it records nothing, and its re-queues ask afresh.

**A replay that cannot be read fails the run.** If the original's decisions cannot be loaded, the
run's first decision fails rather than quietly asking again, because the run was queued to repeat
the original.

**The run is identified by its row.** A run started from a row the scheduler created takes that
row's external id, so the decisions it records and the failure data it carries name the run they
belong to. Before this, such a run kept the random id its train instance was constructed with.

**Changing the queue API without breaking it.** `ITrainExecutionService` gained an overload that
takes `QueueTrainOptions`, with a default implementation that refuses a replay rather than
dropping it. `QueueTrainInput` gained an init property, so its constructor is unchanged.

## Exemplars

**Enforced elsewhere:** `DecisionRecordingTests` in Trax.Effect pins that a run records each
decision and the track it took against its metadata, that a decision not followed records why,
that a run that fails after deciding still records, that a run naming an earlier one replays its
decisions without asking, that a replay of a run with nothing recorded asks afresh, and that a
run started from a row takes the row's external id. `QueueInputTests` in Trax.Mediator pins that
the options overload stores the link and that an implementation predating it refuses a replay.
`OperationsServiceTests` and `JobDispatcherTrainTests` in Trax.Scheduler pin the link through the
enqueue and onto the dispatched run's metadata. `OperationsQueriesTests` in Trax.Api and
`MetadataRequeueTrustedScopeTests` in Trax.Dashboard pin that both re-queue surfaces pass the
original run's id.

Not covered: a train whose chain changed between the original run and its re-queue replays the
questions the two have in common and asks the new ones afresh; nothing flags that the replayed
run took a path the original chain could not have.

## Changelog

- **2026-10-01**: Recorded.
