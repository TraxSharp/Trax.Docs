---
layout: default
title: Dead Letters & Cleanup
description: How a manifest dead-letters after MaxRetries within its failure window, and how to requeue or acknowledge dead letters, configure backoff and clean up metadata.
parent: Scheduling
nav_order: 3
---

# Dead Letters, Monitoring & Cleanup

## Handling Dead Letters

When a job's counted failures exceed `MaxRetries` (the retries allowed after the first run, so the default of 3 dead-letters on the fourth failure), it enters the dead letter queue with status `AwaitingIntervention`. Failures count only while they started within the manifest's failure window (`FailureCountWindow`, default 24 hours, unless the manifest states its own `FailureWindow`), so failures spread over weeks do not dead-letter a manifest. The ManifestManager will skip these manifests until they're resolved.

To resolve a dead letter, use the **Dashboard UI**, the **GraphQL API**, or the **ITraxScheduler** service directly.

### Via Dashboard

Navigate to **Data > Dead Letters** and click the visibility icon on any row. The dead letter detail page shows full context, the dead letter reason, manifest configuration, the most recent failure's stack trace, and a history of all failed runs.

Two actions are available while the dead letter is in `AwaitingIntervention` status:

- **Re-queue**: Creates a new WorkQueue entry from the manifest's properties and marks the dead letter as `Retried`. The run [replays the failed run's decisions](#retries-replay-decisions) when that is sound.
- **Re-queue, Ask Afresh**: The same, except the run asks its deciders afresh
- **Acknowledge**: Prompts for a resolution note and marks the dead letter as `Acknowledged`

#### Batch Operations

The dead letters list page supports batch operations for resolving multiple dead letters at once:

- **Requeue All / Acknowledge All**: Resolves every `AwaitingIntervention` dead letter in a single operation
- **Requeue Selected / Acknowledge Selected**: Use the checkboxes to select specific dead letters, then resolve just those
- **Requeue All, Ask Afresh / Requeue Selected, Ask Afresh**: Requeue as above, with each run asking its deciders afresh

### Via GraphQL

Dead letter operations live under `operations.deadLetters`. The `operations` namespace is opt-in (off by default), wired via [`ExposeOperationQueries()`](/docs/sdk-reference/graphql-api/add-trax-graphql) and [`ExposeOperationMutations()`](/docs/sdk-reference/graphql-api/add-trax-graphql) on the `AddTraxGraphQL` builder. Once enabled:

```graphql
# Requeue a single dead letter
mutation {
  operations {
    deadLetters {
      requeueDeadLetter(id: 42) {
        success
        workQueueId
        message
      }
    }
  }
}

# Acknowledge a single dead letter
mutation {
  operations {
    deadLetters {
      acknowledgeDeadLetter(id: 42, note: "Root cause fixed") {
        success
        message
      }
    }
  }
}

# Batch requeue by IDs
mutation {
  operations {
    deadLetters {
      requeueDeadLetters(ids: [42, 43, 44]) {
        count
        message
      }
    }
  }
}

# Requeue all awaiting dead letters
mutation {
  operations {
    deadLetters {
      requeueAllDeadLetters {
        count
        message
      }
    }
  }
}

# Requeue so the run asks its deciders afresh
mutation {
  operations {
    deadLetters {
      requeueDeadLetter(id: 42, askAfresh: true) {
        success
        message
      }
    }
  }
}
```

`requeueDeadLetter`, `requeueDeadLetters` and `requeueAllDeadLetters` take `askAfresh` (default
`false`), as `triggerManifest` and `triggerManifestDelayed` do; see
[Retries replay decisions](#retries-replay-decisions).

Query dead letters with optional status filtering:

```graphql
query {
  operations {
    deadLetters {
      getDeadLetters(status: AWAITING_INTERVENTION, take: 10) {
        items {
          id
          manifestName
          status
          reason
          deadLetteredAt
        }
        totalCount
      }
    }
  }
}
```

### Via ITraxScheduler

All dead letter operations are available programmatically through `ITraxScheduler`:

```csharp
// Single operations
var result = await scheduler.RequeueDeadLetterAsync(deadLetterId);
var result = await scheduler.AcknowledgeDeadLetterAsync(deadLetterId, "Root cause fixed");

// Batch by IDs
var result = await scheduler.RequeueDeadLettersAsync(new long[] { 1, 2, 3 });
var result = await scheduler.AcknowledgeDeadLettersAsync(new long[] { 1, 2, 3 }, "Batch fix");

// Resolve all
var result = await scheduler.RequeueAllDeadLettersAsync();
var result = await scheduler.AcknowledgeAllDeadLettersAsync("Mass acknowledge");

// Requeue so the run asks its deciders afresh instead of replaying the failed run's decisions
var result = await scheduler.RequeueDeadLetterAsync(deadLetterId, askAfresh: true);
var result = await scheduler.RequeueDeadLettersAsync(new long[] { 1, 2, 3 }, askAfresh: true);
var result = await scheduler.RequeueAllDeadLettersAsync(askAfresh: true);
```

A requeue [replays the decisions](#retries-replay-decisions) of the manifest's failed run when that
is sound, as an automatic retry does. Pass `askAfresh: true` to queue the run to ask its deciders
again instead.

The batch methods take 1 to `OperationsService.MaxBatchSize` (1000) ids, the limit every operations-surface batch has. An empty list or a longer one is refused: the result counts nothing and its message says why. `RequeueAllDeadLettersAsync` reads and requeues a page of manifests at a time and sums the counts, so a large backlog is never loaded at once; every dead letter of one manifest is in the same page, so they still fold into one entry.

### Failure Counter Reset

Resolving a dead letter (either action) resets the manifest's failure counter. The ManifestManager only counts failures that occurred **after** the most recent resolution (and inside `FailureCountWindow`) when comparing against `MaxRetries`. This means a retried manifest starts fresh, it won't be immediately re-dead-lettered based on the same failures that triggered the original dead letter.

### One Queued Entry per Manifest

A manifest can have only one queued work queue entry at a time (a unique index enforces it), so a requeue never creates a second one:

- **A single requeue** whose manifest already has a queued entry fails with a message naming that entry, and the dead letter stays `AwaitingIntervention`.
- **A batch or "all" requeue** skips each dead letter whose manifest already has a queued entry, leaving it `AwaitingIntervention`. Dead letters that share a manifest are folded into one entry and all marked `Retried`, since a requeue runs the manifest's own properties and each would queue the same work. Each resolution note names the entry; the newest dead letter carries the entry's `DeadLetterId`.

`count` is the number of dead letters resolved, and `message` says how many were folded and how many were skipped, for example `"3 dead letter(s) requeued; 1 folded into another dead letter's entry for the same manifest; 2 skipped because their manifest already has a queued entry."` A skipped dead letter can be requeued once the queued entry has been dispatched.

### Acknowledging in Bulk

On a relational provider, **Acknowledge All** and **Acknowledge Selected** resolve their dead letters with a single `UPDATE`, so acknowledging a backlog of tens of thousands takes a fraction of a second. The InMemory provider has no set-based update, so there each row is loaded and saved.

### RetryMetadataId Linking

When a dead letter is requeued, the new WorkQueue entry carries a `DeadLetterId` reference. When the JobDispatcher creates a Metadata record for that entry, it automatically links the `RetryMetadataId` back on the dead letter. This creates a traceable chain from the original failure through the dead letter to the retry execution.

### Concurrency Safety

All dead letter operations filter by `status = 'awaiting_intervention'` at query time. If two users resolve the same dead letter simultaneously, the second operation sees no matching record and returns a "not found or already resolved" result. No duplicate work queue entries are created. The check for a manifest's existing queued entry and the insert are separate statements, so a concurrent requeue or the ManifestManager can queue that manifest in between. The unique index refuses the second entry; a batch or "all" requeue then rereads and retries, skipping that manifest, and a single requeue reports it as already queued. Two operators pressing **Requeue All** at once both succeed and leave one queued entry per manifest.

## Retry Delay & Backoff

When a manifest's latest finished run failed and its counted failures have not exceeded `MaxRetries`, the scheduler delays the next run, the retry, by an exponential backoff. `failureCount` is the number of failures inside the manifest's failure window (its `FailureWindow`, or the scheduler's `FailureCountWindow`) since the latest resolved dead letter, so a failure older than the window no longer lengthens the delay:

```
delay = min(DefaultRetryDelay * RetryBackoffMultiplier ^ (failureCount - 1), MaxRetryDelay)
```

With defaults (`DefaultRetryDelay: 5m`, `RetryBackoffMultiplier: 2.0`, `MaxRetryDelay: 1h`):

| Failure # | Delay |
|-----------|-------|
| 1 | 5 minutes |
| 2 | 10 minutes |
| 3 | 20 minutes |
| 4 | 40 minutes |
| 5+ | 1 hour (capped) |

Only a retry waits. Once a run succeeds or is cancelled, the next occurrence runs on time, however many failures are still inside the window; those failures still set the length of the next retry's delay and still count toward the dead letter. A requeued dispatch attempt is not a finished run and does not count either way.

The delay is implemented by setting `ScheduledAt` on the WorkQueue entry. The JobDispatcher skips entries where `ScheduledAt > now`, so the retry won't be dispatched until the delay has elapsed.

Configure via the scheduler builder:

```csharp
.AddScheduler(scheduler => scheduler
    .DefaultRetryDelay(TimeSpan.FromMinutes(5))
    .RetryBackoffMultiplier(2.0)
    .MaxRetryDelay(TimeSpan.FromHours(1))
)
```

## Retries replay decisions

A retry exists because something after a decision failed: a tool step threw, a database timed out.
Trax has no per-junction retry, so a retry runs the chain from its first junction, and asking a
model again costs a call per question and can be answered differently. So when a manifest's run
fails, its retry, queued by the ManifestManager, and a requeue of its dead letter replay the
[decisions](/docs/effect/decisions#re-queued-and-retried-runs-replay-their-decisions) the failed
run recorded: the retry takes the tracks the failed run's deciders chose instead of asking them
again. Only the answers are replayed. Every ordinary junction runs again, side effects included.

The scheduler reads the run to replay from the database, never from a caller: it is the manifest's
latest finished run, and only when the next run is a retry of it, that is, the run failed and its
failure still counts toward the retries (the same test that applies the backoff). A failure that an
acknowledged dead letter or the failure window has set aside is not retried, so the next
occurrence asks afresh. It links the retry only when all of these hold:

| Check | Otherwise |
|---|---|
| The manifest replays decisions on retry (`ReplayDecisionsOnRetry`, on by default) | Asked afresh |
| The failed run asked its deciders itself, rather than replaying another run's answers | Asked afresh |
| Nothing else replays the failed run: no run in any state (queued, running or finished) and no queued entry | Asked afresh |
| The failed run is inside its train's metadata retention | Asked afresh |
| For a dependent manifest, its parent has not succeeded again since the failed run started; a new parent success fires a new run, not a retry | Asked afresh |
| The failed run is a run of the manifest's train, recorded its decisions and acted on at least one | Asked afresh |
| The failed run was queued by this manifest, with no subject key, and with exactly the input and input type the retry is queued with | Asked afresh |

So a retry replays at most once in a row: answers that were replayed into a failure are not
replayed again, and the next retry asks afresh. A manifest edited between the failure and the retry
asks afresh, because the answers were given about the old input. Asking afresh is never an error,
and a lookup that fails is logged and asks afresh rather than holding up the retry. The retention
check assumes every host that runs metadata cleanup uses the same retention, as one deployment's
hosts do.

A link can still become impossible to honour after it is written: the failed run can be deleted
before the retry runs, or the retry can land on a host that does not record decisions. A manifest's
run then logs a warning and asks afresh, rather than failing as a manual requeue would.

Within the replay, each answer is still checked one by one: it replays only into a question asked
the same way, about a state that hashes the same, and only while it is younger than
[`ReplayAnswersFor`](/docs/sdk-reference/configuration/add-decision-recording#decisionrecordingoptions)
(24 hours by default). An answer that fails any of these is asked afresh. The InMemory provider
does not replay retries, because its ManifestManager dispatches without work queue entries to
compare inputs against. Replay needs [`AddDecisionRecording()`](/docs/sdk-reference/configuration/add-decision-recording)
on the hosts that run the manifest's train.

### Asking afresh on purpose

A manifest whose questions should always be answered on current information turns replay off:

```csharp
scheduler.Schedule<IScoreLeadsTrain>(
    "score-leads",
    new ScoreLeadsInput(),
    Every.Minutes(30),
    options => options.ReplayDecisionsOnRetry(false));
```

The flag is stored on the manifest as `replay_decisions_on_retry`, so every scheduler host reads the
same value. Stated in code, it is written on every seed, so `ReplayDecisionsOnRetry(true)` in code
turns replay back on at each restart over an operator's runtime opt-out, as every stated setting
does. Where operators manage the flag at runtime, leave it out of code. Left unstated, a new
manifest replays and an existing one keeps its value. [`IOperationsService.SetManifestsReplayDecisionsOnRetryAsync`](/docs/sdk-reference/scheduler-api/i-operations-service#batch-actions)
sets it at runtime. Turning it off reaches a retry already queued: the write clears the link on the
manifest's queued entry, and the dispatcher checks the flag again when it claims an entry. An
opt-out committed in the instant between that check and the run's start still lets that one run
replay.

For a single occasion, the dead-letter requeues and `TriggerAsync` take `askAfresh`. A requeue
asked afresh queues no link, and a trigger asked afresh clears the link of the queued retry it
releases. `Trax.Scheduler/docs/adr/0017` records why retries replay, and why only once.

## Monitoring

The **Trax.Core Dashboard** at `/trax/data/dead-letters` provides a real-time view of all dead letters with status badges and links to detail pages. The dead letter detail page surfaces the full failure context, stack traces, inputs, and execution history, so operators can make informed retry/acknowledge decisions without writing queries.

The built-in local workers execute JobRunner jobs using PostgreSQL's `background_job` table. Worker health and job status can be monitored via the Trax Dashboard at `/trax`.

For train-level details, query the `Metadata` table:

```csharp
// Recent failures for a manifest
var failures = await context.Metadatas
    .Where(m => m.ManifestId == manifestId && m.TrainState == TrainState.Failed)
    .OrderByDescending(m => m.StartTime)
    .Take(10)
    .ToListAsync();
```

## Metadata Cleanup

System trains like `ManifestManagerTrain` run frequently (every 5 seconds by default), generating metadata rows that have no long-term value. The metadata cleanup service automatically purges old entries to keep the database clean.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│        MetadataCleanupPollingService (BackgroundService)         │
│            Polls on CleanupInterval (default: 1 minute)         │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  MetadataCleanupTrain                         │
│                                                                  │
│  DeleteExpiredMetadataJunction:                                      │
│    1. Find metadata matching whitelist + older than retention    │
│    2. Only terminal states (Completed / Failed / Cancelled)      │
│    3. Delete associated work queue entries (FK safety)            │
│    4. Delete associated log entries (FK safety)                  │
│    5. Delete metadata rows                                       │
└─────────────────────────────────────────────────────────────────┘
```

The cleanup only targets metadata in **terminal states** (Completed, Failed, or Cancelled). Pending and InProgress metadata is never deleted, regardless of age. Associated work queue entries and log entries are deleted first to avoid foreign key constraint violations.

Deletion uses EF Core's `ExecuteDeleteAsync` for efficient single-statement SQL. No entities are loaded into memory.

### Enabling Cleanup

Add `.AddMetadataCleanup()` to your scheduler configuration. By default this cleans up the internal scheduler trains (`JobDispatcher`, `ManifestManager`, `MetadataCleanup`, `DeadLetterCleanup`, `JobRunner`) metadata older than 30 minutes, checking every minute.

See [MetadataCleanup](/docs/scheduler/admin-trains/metadata-cleanup) for details on how the cleanup train operates internally.

### What Gets Deleted

A metadata row is deleted when **all** of these conditions are true:

1. Its `Name` matches a train in the whitelist
2. Its `StartTime` is older than the retention period
3. Its `TrainState` is `Completed`, `Failed`, or `Cancelled`

Any work queue entries and log entries associated with deleted metadata are also removed.

Cancelled trains are treated as terminal, they are eligible for cleanup but are **not retried** and **do not create dead letters**. Cancellation is an explicit operator action, not a transient failure. A run that stopped because something inside it gave up, such as an `HttpClient` timeout, was not cancelled: it is recorded `Failed` and classified `Transient`, so it counts toward `MaxRetries` like any other failure.

## Dead Letter Auto-Purge

Resolved dead letters (Retried or Acknowledged) are automatically purged after the configured retention period. This is enabled by default.

The `DeadLetterCleanupTrain` runs on its own polling interval (default: 1 hour) and deletes resolved dead letters where `ResolvedAt` is older than `DeadLetterRetentionPeriod` (default: 30 days). Dead letters in `AwaitingIntervention` status are never deleted. Nor is a retried dead letter whose requeued work queue entry is still `Queued` (a paused dispatcher, or a group at its limit): deleting the dead letter would delete that retry before it ran. It is purged on a later run, once the retry has left the queue.

Configure via the scheduler builder:

```csharp
.AddScheduler(scheduler => scheduler
    // Disable auto-purge (dead letters accumulate forever)
    .AutoPurgeDeadLetters(false)

    // Or customize retention
    .DeadLetterRetentionPeriod(TimeSpan.FromDays(90))
)
```

Both settings can also be changed at runtime, from the dashboard's Server Settings page or the [`updateScheduler`](/docs/sdk-reference/graphql-api/mutations#config-nested-namespace) mutation. The purge reads them on every run, so turning auto-purge off keeps resolved dead letters from the next run on, without a restart, and the cleanup service stays registered so it can be turned back on the same way.

Both settings delete data, so when the builder states them and a saved setting does too, they fail closed rather than letting the saved value win:

| Code | Saved | Result |
|------|-------|--------|
| `AutoPurgeDeadLetters(false)` | `true` | No purge |
| `AutoPurgeDeadLetters(true)` or unstated | `false` | No purge |
| `DeadLetterRetentionPeriod(90 days)` | 7 days | 90 days |
| `DeadLetterRetentionPeriod(7 days)` | 90 days | 90 days |

When the builder does not state a setting, a saved value replaces the default as any other saved setting does. In every case where the saved value differs from the host's code value, the scheduler logs a warning naming the setting, both values and the one it runs with.

## Testing

For integration tests, use `UseInMemory()` and the scheduler will automatically use its in-memory job submitter:

```csharp
services.AddTrax(trax => trax
    .AddEffects(effects => effects.UseInMemory())
    .AddMediator(typeof(Program).Assembly)
    .AddScheduler()
);
```

Jobs execute inline, so tests are fast and don't need database infrastructure.

## SDK Reference

> [AddMetadataCleanup](/docs/sdk-reference/scheduler-api/add-metadata-cleanup) | [AddDecisionRecording](/docs/sdk-reference/configuration/add-decision-recording) | [Schedule](/docs/sdk-reference/scheduler-api/schedule) | [AddScheduler](/docs/sdk-reference/scheduler-api/add-scheduler) | [ManifestManagement](/docs/sdk-reference/scheduler-api/manifest-management) | [ITraxScheduler](/docs/sdk-reference/scheduler-api/i-trax-scheduler#dead-letters)
