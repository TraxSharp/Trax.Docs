---
layout: default
title: AddSystemOneDecider
description: Reference for AddSystemOneDecider, which answers train decisions through a typed decision model that speaks the System One format, such as Jev or Laya.
parent: Configuration
grand_parent: SDK Reference
nav_order: 23
---

# AddSystemOneDecider

Answers every train's [decisions](/docs/core/decisions) through a typed decision model that speaks
the System One request format (Jev, d1, Laya, Kev and others), by registering a
`SystemOneDecider` as the `IDecider`. For Nimble, use
[AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider), which fills in its
endpoints, models and limits. See
[Typed decision models](/docs/effect/decisions#other-typed-decision-models).

## Signature

```csharp
public static TBuilder AddSystemOneDecider<TBuilder>(
    this TBuilder configurationBuilder,
    Action<SystemOneOptions> configure
)
    where TBuilder : TraxEffectBuilder
```

## SystemOneOptions

| Property | Type | Default | Description |
|---|---|---|---|
| `Endpoint` | `Uri?` | required | The model's endpoint, such as `https://api.typesafe.ai/v1/systemone`. Must be HTTPS unless it is a loopback address. |
| `Model` | `string?` | required | The model pinned to a version, such as `jev-1.13.0`. A floating alias (ending `latest`, or with no version digits) is refused. |
| `AllowFloatingModel` | `bool` | `false` | Accepts a floating alias. |
| `ApiKey` | `string?` | `null` | Sent as a bearer token. Never logged. |
| `AttemptTimeout` | `TimeSpan` | 10 seconds | How long one attempt may take before it is abandoned and retried. |
| `MaxAttempts` | `int` | `3` | Attempts per request, counting the first. |
| `RetryDelay` | `TimeSpan` | 500 ms | The wait before the first retry, doubling after each, unless the model sends `Retry-After`. Capped at 30 seconds. |
| `MaxOptions` | `int` | `255` | The most options or levels one question may offer, from 2 to 255. A question with more is refused before it is sent. |
| `MaxQuestions` | `int` | `64` | The most questions one request may carry. A `Decide` asking more is refused before it is sent. |
| `MaxConcurrentRequests` | `int?` | `null` | Requests in flight at once, or no limit. A request over the limit waits for a slot, which does not count against its `AttemptTimeout`. |

## Returns

`TBuilder`, the same builder type that was passed in.

## Example

```csharp
services.AddTrax(trax => trax
    .AddEffects(effects => effects
        .UsePostgres(connectionString)
        .AddDecisionRecording()
        .AddSystemOneDecider(o =>
        {
            o.Endpoint = new Uri("http://localhost:8080/v1/systemone");   // a self-hosted model
            o.Model = "laya-1.2.0";
        })
    )
);
```

## Failures

| What the model did | Retried | Run fails with | Failure class |
|---|---|---|---|
| 408, 429 or 5xx (including Nimble's 529, busy), no answer in time, or could not be reached | yes, until `MaxAttempts` | `DecisionServiceException` | `Transient` |
| Any other 4xx (bad criteria, a bad key, 402 for no credit left) | no | `DecisionServiceException` | `Permanent` |
| Too many questions or options, before anything is sent | no | `DecisionServiceException` | `Permanent` |
| An answer it cannot read | no | the train's unanswered-question failure | `Permanent` |

Cancelling the run cancels the request and is not retried. A failure's message carries the model's
`detail` and `request_id` when its error body has them, as Nimble's does; the request id is what the
provider asks for when a failure is reported.

## Remarks

- The options are checked when this is called, so unusable settings stop the host from starting.
- Registers the decider as `SystemOneDecider` and as `IDecider`. To escalate what it is unsure of,
  register a `CascadingDecider` as the `IDecider` after it, built from `SystemOneDecider`.

## Package

```
dotnet add package Trax.Effect.Decisions.SystemOne
```
