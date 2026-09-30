---
layout: default
title: Effects
parent: State Machine API
grand_parent: SDK Reference
nav_order: 7
---

# Effects

A machine's one consequential transition (send a letter, charge a card, provision a resource) binds an
`ISnapshotEffect`. Trax runs it exactly once when that transition is sent, records its receipt in the
snapshot, and never runs it twice, even across a crash and retry.

```csharp
public interface ISnapshotEffect
{
    Task<string> Run(Snapshot snapshot, CancellationToken cancellationToken = default);
}
```

`Run` performs the side effect and returns a receipt, the downstream id (a message-log id, a charge id) that
proves it happened. Throwing means the effect did not complete: the claim is released, the transition is not
applied, and the client can retry from the same state.

An `OperationCanceledException` is different: a charge can land and its response time out, so a cancellation
does not say whether the effect happened. The claim stays in flight until its lease passes, and a send before then
is refused as `effect-in-progress` instead of running the effect again. The send reports it as `delivery-failed`
unless the request itself was cancelled. The send passes `Run` `CancellationToken.None`, not the request's token:
a client that disconnects mid-charge does not cancel the charge. Bound a slow downstream call with its own
timeout, and never after the irreversible step.

## Binding it

Bind the effect inline on the transition with [`RunsOnce<TEffect>`](/docs/sdk-reference/statemachine-api/fluent-authoring),
and implement it in the host:

```csharp
m.In(CheckoutState.Review)
    .On(CheckoutTrigger.Pay)
        .When(Field((ReviewContext c) => c.Items).CountAtLeast(1))
        .RunsOnce<ICharge>()                       // keyPrefix defaults to "checkout:Pay"
        .Reduce(Set((PaidContext p) => p.Receipt).FromInput((PayInput i) => i.Receipt))
        .To(CheckoutState.Paid);
```

```csharp
public interface ICharge : ISnapshotEffect;

public sealed class StripeCharge(IPaymentGateway gateway) : ICharge
{
    public async Task<string> Run(Snapshot snapshot, CancellationToken ct)
    {
        var chargeId = await gateway.Charge(snapshot.Context);
        return chargeId;                            // becomes the receipt
    }
}
```

Register the implementation like any service; the machine resolves `TEffect` from DI, so nothing is wired in
the composition root by hand:

```csharp
services.AddScoped<ICharge, StripeCharge>();
```

## Exactly-once and the receipt

The effect runs through the persistence layer's idempotent path: a claim is taken before the effect, held
under a lease with a fence token, and a crash mid-flight replays without re-running a completed effect. The
key is `{keyPrefix}:{userKey}:{id}`, so it is scoped per draft per user. A draft deleted by the draft TTL releases
the key, and so does a reset to the initial state once the effect's outcome is settled on the draft: a reset while
the effect runs, or after a receipt that never reached the draft, keeps the claim, and the next send replays its
receipt. See [what each path may write](/docs/sdk-reference/statemachine-api/persistence-ports#what-each-path-may-write).

Once the effect has returned, its receipt is recorded and the draft advanced on a token the request cannot
cancel. A client that disconnects right after a charge still leaves the draft showing the charge, and the next
send replays it instead of charging again. The receipt is recorded only on the draft exactly as the effect loaded
it: if the draft was saved, reset or advanced while the effect ran, the send reports `conflict` and records
nothing, and the claim keeps the receipt.

The claim also records a fingerprint of the content the effect ran on: the SHA-256 of the draft's canonical wire,
`SnapshotFingerprint.Of(service.Serialize(snapshot))`. A later send replays the receipt only onto a draft whose
content has that fingerprint. If the draft now holds anything else, the send is refused as `draft-changed`: the
receipt is not recorded and the effect does not run again. Restoring the content the effect ran on, for example by
autosaving the snapshot the client sent, makes the next send replay the receipt, so a client keeps that snapshot
until its send settles. A claim recorded before the fingerprint existed has none and replays onto whatever the
draft holds.

The transition itself is fired only by the send. An advance of its trigger is refused as `effect-bound`, and an
autosave cannot put a draft into its destination state; see
[what each path may write](/docs/sdk-reference/statemachine-api/persistence-ports#what-each-path-may-write).

The receipt `Run` returns is handed to the transition's reducer as `input["receipt"]`, which is how the send
gets recorded in the destination context. It must be non-empty. The ledger reads a claim with no receipt as
still in flight, so a `null` or `""` receipt is treated as a failed effect: the claim is released,
`IdempotentEffect.RunOnce` throws `InvalidOperationException`, and the send returns `delivery-failed` with the
draft unchanged, exactly as if `Run` had thrown. A guard on the same edge can require it (so the transition only
completes once the effect has produced a receipt).
