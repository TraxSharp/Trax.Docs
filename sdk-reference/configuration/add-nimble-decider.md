---
layout: default
title: AddNimbleDecider
description: Reference for AddNimbleDecider, which answers train decisions with Nimble, Bespoke Labs' open-weights decision model, on a local Ollama or hosted.
parent: Configuration
grand_parent: SDK Reference
nav_order: 22
---

# AddNimbleDecider

Answers every train's [decisions](/docs/core/decisions) with Nimble, Bespoke Labs' open-weights
typed decision model: on a local Ollama by default, or on Bespoke's hosted API when an API key is
given. Registers a `SystemOneDecider` as the `IDecider`. See
[Nimble](/docs/effect/decisions#nimble).

## Signature

```csharp
public static TBuilder AddNimbleDecider<TBuilder>(
    this TBuilder configurationBuilder,
    Action<NimbleOptions>? configure = null
)
    where TBuilder : TraxEffectBuilder
```

## NimbleOptions

| Property | Type | Default | Description |
|---|---|---|---|
| `ApiKey` | `string?` | `null` | A Bespoke Labs API key. Setting it selects the hosted API; leaving it null selects a local Ollama. Never logged. |
| `Endpoint` | `Uri?` | local: `http://localhost:11434/v1/systemone`; hosted: `https://api.bespokelabs.ai/v1/systemone` | Another endpoint, such as an Ollama on another machine. Must be HTTPS unless it is a loopback address. |
| `Model` | `string?` | local: `nimble:9b`; hosted: `nimble-v3` | The model, pinned. `nimble-latest`, `nimble:latest` and a bare `nimble` are refused. |
| `MaxConcurrentRequests` | `int?` | local: no limit; hosted: `8` | Requests in flight at once. Bespoke allows 8 per organisation; more wait for a slot instead of being answered 429. |
| `AttemptTimeout` | `TimeSpan` | 30 seconds | How long one attempt may take before it is abandoned and retried. |
| `MaxAttempts` | `int` | `3` | Attempts per request, counting the first. |
| `RetryDelay` | `TimeSpan` | 500 ms | The wait before the first retry, doubling after each, unless the model sends `Retry-After`. |

A request carries at most 64 questions and a question at most 255 options or levels, Nimble's own
limits. The constants `NimbleOptions.LocalEndpoint`, `HostedEndpoint`, `LocalModel`, `HostedModel`
and `HostedConcurrentRequests` hold the defaults.

## Returns

`TBuilder`, the same builder type that was passed in.

## Examples

```csharp
// Nimble on this machine. Run `ollama pull nimble:9b` first.
services.AddTrax(trax => trax
    .AddEffects(effects => effects
        .UsePostgres(connectionString)
        .AddDecisionRecording()
        .AddNimbleDecider()
    )
);

// Bespoke's hosted Nimble.
effects.AddNimbleDecider(o => o.ApiKey = configuration["Nimble:ApiKey"]);

// One GPU machine serving Nimble to several hosts.
effects.AddNimbleDecider(o => o.Endpoint = new Uri("https://nimble.internal.example/v1/systemone"));
```

## Remarks

- The options are checked when this is called, so an unpinned model or plain HTTP to another
  machine stops the host from starting.
- Retries, failure classes and what an unreadable answer does are those of
  [AddSystemOneDecider](/docs/sdk-reference/configuration/add-system-one-decider#failures).
- To put Nimble in front of a larger model, register a `CascadingDecider` as the `IDecider` after
  this, built from `SystemOneDecider`.

## Package

```
dotnet add package Trax.Effect.Decisions.SystemOne
```
