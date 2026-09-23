---
authors: [Theauxm]
repos: [effect, mediator, scheduler, api, dashboard]
areas: [platform, data-model]
status: accepted
---

# Queued work for one subject runs one at a time

A train can name the subject its queued work touches with `QueueSubjectKey`. Dispatch does not
claim an entry while another entry for the same subject has a run in flight, which removes the
lost update two concurrent writes to one record would otherwise produce. The check lives in the
claim, behind a per-subject advisory lock, because "in flight" is a fact about another table.

## Status

**Accepted.**

## Considered options

**A unique partial index.** It cannot express the rule. A subject is busy while a dispatched
entry's run has not reached a terminal state, which is a join to `trax.metadata`, not a property
of the work queue row.

**Row locking alone.** Two entries for one subject are two rows, so `FOR UPDATE SKIP LOCKED` does
not make them contend, and while both are still queued neither sees a dispatched sibling to
refuse itself. The claim therefore takes `pg_advisory_xact_lock` in its two-key form under a
fixed class key, which Postgres keeps apart from the single-key space the leader lock and a
consumer's own locks use.

**Strict enqueue order within a subject.** Rejected in favour of dispatch order: priority, then
age. Priority should still mean something, and strict order across several dispatchers would need
a per-subject queue.

## Consequences

The key is an opaque string compared exactly across every train, so two trains returning the same
key serialize against each other. It is refused when empty or longer than 512 characters. The
guarantee holds within the stale-run window: when a reaper fails a run (pending longer than
`StalePendingTimeout`, or in progress longer than `StaleInProgressTimeout`), the subject is
released even if that run is still working. A synchronous run through the mediator does not
consult the key, and neither does a dormant dependent a parent train activates: its entry is built
by the scheduler with input the parent chose at runtime, and carries no subject. The API exposes
the key as `subjectKey` on work queue reads.

Every dispatcher must be upgraded before any train overrides `QueueSubjectKey`. A dispatcher from
before this decision claims without the subject check or lock, so during a rolling deploy it can
run a second entry for a subject that already has one in flight, and the guarantee does not hold
until the last old dispatcher is gone. Dispatch drops busy subjects and
duplicate siblings from its candidates, so entries the claim would refuse do not use up
`MaxActiveJobs`.

## Exemplars

**Enforced elsewhere:** `SubjectKeySerializationTests` in Trax.Scheduler (one run per subject,
release on completion and on reaping, the two-transaction claim race, and capacity on both load
paths) and `SubjectKeyTests` in Trax.Mediator (where the key comes from, and the empty and length
limits).

Not covered: ordering with more than one dispatcher, and the reaper releasing a subject whose run is
still working, which is the documented limit rather than something a test can rule out.

## Changelog

- **2026-09-23**: Recorded.
