---
layout: default
title: State Machine
description: "The StateMachine sample: two fluent machines behind the generic stateMachine mutations, a forward migration, an exactly-once charge and a server-owned total."
parent: Samples & Deployment
nav_order: 10
---

# State Machine

> NO WARRANTY. Trax auth is plumbing, not a security product. You are solely responsible for securing systems that use it. See [API Security](/docs/api-security).

`samples/StateMachine` in [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) is a GraphQL host
over two snapshot state machines, authored with the fluent API and driven through the four generic
`stateMachine` mutations, plus a React app (`web/`) that drives them from a browser.

| Machine | Shape | What it shows |
|---|---|---|
| `turnstile` | `Locked ⇄ Unlocked` | Structure only: guards, reducers, per-state invariants. No effect. |
| `checkout` | `Cart → Review → Paid`, version 2 | A committed state, one irreversible charge run exactly once, a forward migration from v1, and a total the server checks |

## What it proves

| Feature | Where |
|---|---|
| One-line discovery: `AddStateMachines(assembly)` before `AddMediator` | `Api/Program.cs` |
| The two bindings a host supplies: `ISnapshotPrincipal` (whose draft) and the effect (`ICharge`) | `Api/Program.cs`, `SnapshotPrincipal.cs` |
| An effect that runs once per intent, however often `sendSnapshot` is retried | `Machines.cs`, `RunsOnce<ICharge>` |
| A v1 draft upgraded to v2 on load by `MigrateFrom(1, ...)` | `CheckoutMachineTests` |
| Server authority over what a client autosaves: a draft whose total disagrees with its items is refused | `CheckoutTotalAuthorityTests` |

## Run

From the `Trax.Samples` root:

```bash
docker compose up -d       # Postgres on 5432
dotnet run --project samples/StateMachine/Trax.Samples.StateMachine.Api
```

The host listens on <http://localhost:5220> in Development (`Properties/launchSettings.json`), the
only environment that registers the demo keys `alice-key-do-not-use-in-production` and
`bob-key-do-not-use-in-production`. The `snapshot_draft` and `effect_claim` tables come from the Trax
Postgres provider's own migrations; the sample writes no schema.

For the browser app, start the host, then:

```bash
cd samples/StateMachine/web
npm install
npm run dev                # http://localhost:5173
```

## Try it

Send `X-Api-Key: alice-key-do-not-use-in-production` with each request (Nitro at
<http://localhost:5220/trax/graphql>, or curl).

```graphql
# What machines are there? (anonymous)
{ discover { stateMachine { listMachines { machines { name hasEffect } } } } }

# Save a checkout draft at Review. The total must be 999 cents per item.
mutation {
  dispatch { stateMachine { saveSnapshot(input: {
    machine: "checkout",
    id: "11111111-1111-1111-1111-111111111111",
    snapshot: "{\"machine\":\"checkout\",\"version\":2,\"state\":\"Review\",\"context\":{\"items\":[\"book\"],\"receipt\":null,\"total\":999}}"
  }) { output { snapshot problem { code } } } } }
}

# Pay: runs the charge once and moves to Paid
mutation {
  dispatch { stateMachine { sendSnapshot(input: {
    machine: "checkout", id: "11111111-1111-1111-1111-111111111111", requestId: "pay-1"
  }) { output { snapshot problem { code } } } } }
}
```

The `sendSnapshot` answer is the `Paid` snapshot with a `receipt`, and the host logs
`Charged 999 cents for checkout Review -> receipt rcpt_...`. Send it again and the same snapshot comes
back with no second charge. Save a draft with `"items":["book","pen"]` and `"total":1` under another
id and the answer is `problem { code: "invalid-context" }`.

## How it works

### Registration

```csharp
using Trax.Effect.Data.Postgres.Extensions;
using Trax.Effect.Extensions;
using Trax.Effect.Provider.Json.Extensions;
using Trax.Effect.StateMachine.Persistence;
using Trax.Mediator.Extensions;

builder.Services.AddTrax(trax =>
    trax.AddEffects(effects => effects.UsePostgres(connectionString).AddJson())
        .AddStateMachines(typeof(TurnstileMachine).Assembly)   // before AddMediator
        .AddMediator(typeof(TurnstileMachine).Assembly)
);

builder.Services.AddScoped<ISnapshotPrincipal, TraxCallerSnapshotPrincipal>();
builder.Services.AddScoped<ICharge, LoggingCharge>();
builder.Services.AddTraxGraphQL(graphql => graphql);
```

`AddStateMachines` discovers every `Machine<TState, TTrigger>` in the assembly and contributes the four
mutations to the mediator scan, so it must come before `AddMediator`. `TraxCallerSnapshotPrincipal`
maps the authenticated Trax caller to the draft's owner, so Alice and Bob each see their own draft
for the same id. See [AddStateMachines](/docs/sdk-reference/statemachine-api/add-trax-state-machines).

### The server owns the total

`saveSnapshot` is the soft path: the client writes the whole snapshot, context included, and the
server validates and stores it. Anything an effect will act on must therefore be checked by the
state's `Holds`, or the client decides it. The checkout's `total` is the amount a real `ICharge`
would take, so every state holds it to the item price:

```csharp
private static bool TotalMatchesItems(JsonObject ctx) =>
    ctx["total"] is JsonValue total
    && total.GetValueKind() == JsonValueKind.Number
    && total.ToJsonString()
        == ((long)ItemsCount(ctx) * UnitPriceCents).ToString(CultureInfo.InvariantCulture);

m.In(CheckoutState.Review)
    .Holds(ctx =>
        ItemsCount(ctx) > 0 && ReceiptEmpty(ctx) && TotalMatchesItems(ctx)
            ? null
            : "Review: non-empty items, no receipt, total = 999 cents per item."
    )
    .On(CheckoutTrigger.Pay)
    .When((ctx, input) => ItemsCount(ctx) > 0 && Receipt(input) is not null)
    .RunsOnce<ICharge>("checkout:charge")
    .To(CheckoutState.Paid);
```

The charge then reads the amount from the snapshot it is handed, which is the server's stored copy,
already validated. See [State Machines: Persistence](/docs/statemachine#persistence-and-exactly-once-effects).

### The forward migration

Version 2 added `total`. A draft an older host stored at version 1 has none, so it would fail the v2
invariants; `MigrateFrom(1, ...)` backfills it from the item count when the draft is loaded:

```csharp
m.Id("checkout").Version(2).StartsAt(CheckoutState.Cart, Fresh)
    .MigrateFrom(1, (state, ctx) =>
    {
        var next = (JsonObject)ctx.DeepClone();
        next["total"] = ItemsCount(ctx) * UnitPriceCents;
        return new MigrationResult(state, next);
    });
```

See [Migrations](/docs/sdk-reference/statemachine-api/migrations).

## Tests

```bash
dotnet test tests/Trax.Samples.StateMachine.Tests
```

The tests drive both machines through the real draft service over an in-memory store, so they need no
database: the v1 to v2 migration, the guard that makes it necessary, autosave then advance, the
canonical wire format, and `CheckoutTotalAuthorityTests` (a mismatched total is refused, a matching
one is saved, the charge reads the snapshot's total).

## SDK Reference

> [AddStateMachines](/docs/sdk-reference/statemachine-api/add-trax-state-machines) | [Fluent authoring](/docs/sdk-reference/statemachine-api/fluent-authoring) | [Effects](/docs/sdk-reference/statemachine-api/effects) | [Migrations](/docs/sdk-reference/statemachine-api/migrations) | [Persistence ports](/docs/sdk-reference/statemachine-api/persistence-ports)
