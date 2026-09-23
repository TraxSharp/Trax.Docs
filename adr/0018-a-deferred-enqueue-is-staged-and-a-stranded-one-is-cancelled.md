---
authors: [Theauxm]
repos: [effect, mediator, scheduler, api, dashboard]
areas: [platform, data-model]
status: accepted
---

# A deferred enqueue is staged, and a stranded one is cancelled

A train whose `OnQueue` hook writes outside Trax's own data context can set
`DeferQueuePromotion`: its work queue entry is committed unconfirmed, the hook runs, and a second
commit confirms it. Dispatch never claims an unconfirmed entry. An entry a crash leaves
unconfirmed is cancelled by the scheduler once it is older than `StaleStagedEntryTimeout`,
unless the host opts into promoting it.

## Status

**Accepted.**

## Considered options

**An outbox, or one transaction across both writes.** A hook writing to another database cannot
share Trax's transaction, and an outbox would move the consumer's side-effect into Trax's
schema. Staging does not make the two writes atomic, but it makes a failure between them
findable: an unconfirmed entry instead of a side-effect with nothing to consume it.

**Promoting stranded entries by default.** Rejected. A stranded entry is ambiguous: the process
may have died after the hook succeeded, before it ran, or after it threw and before the entry
was removed. Only the first is a mutation that was accepted, and nothing recorded tells them
apart. Cancelling keeps the entry visible so a side-effect the hook may have left can be
reconciled. `PromoteStaleStagedEntries()` exists for hosts whose chains re-check what their
hooks checked and whose hooks are idempotent.

**Rejecting nested enqueues.** The ambient enqueue context was a scoped field that refused to
nest, so two enqueues sharing a scope, such as a Blazor circuit, threw. It flows with the async
call instead: each enqueue sees its own context, and one started from inside a hook gets its own
transaction and hands the outer context back.

## Consequences

Once a hook has returned, confirming the entry does not take the caller's token, and removing
the entry of a hook that threw does not either, for the same reason effect/0005 gives for a
run's outcome: a record that is only written when the caller is still listening is not a record.
`confirmed_at` defaults to `now()` on Postgres so that rows written during a rolling deploy by an
instance that does not know the column are dispatchable. A deferring train's hook has no enqueue
context to join, because its entry is already committed.

## Exemplars

**Enforced elsewhere:** `DeferredPromotionTests` in Trax.Mediator (staging, removal on a throw,
cancellation before and after the hook, the sweeps), `WorkQueuePromotionTests` in Trax.Effect
(the in-memory provider), `ResolveStaleStagedEntriesJunctionTests` in Trax.Scheduler (cancel by
default, promote when opted in), `SubjectKeySerializationTests` in Trax.Scheduler (stranded
staged entries take no dispatch capacity on either load path), and `EnqueueContextAccessorTests`
in Trax.Effect (nesting and concurrency).

Not covered: nothing checks that a deferring train's hook is idempotent, which promotion relies
on, or that `StaleStagedEntryTimeout` is longer than the slowest hook. The rolling-deploy default
is asserted only by the migration itself.

## Changelog

- **2026-09-23**: Recorded.
