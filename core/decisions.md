---
layout: default
title: Decisions
description: How a chain declares several tracks and lets a decider, from a typed decision model to a rule table, choose one per run, with every track verified at startup.
parent: Core
nav_order: 6
---

# Decisions

A train can take different junctions on different runs, along tracks its chain declares. A
`Switch`, `Gate` or `Scale` names every track it can send the train down; at run time a
**decider** answers a typed question about a value in Memory, and the train takes the matching
track. Every track is part of the declaration, so the host's
[startup check](/docs/core/trains-and-junctions#the-host-checks-every-chain-before-it-serves-traffic)
verifies all of them before the first run.

```csharp
protected override Task<Either<Exception, Resolution>> Junctions() =>
    Chain<ParseTicket>()
        .Switch<Ticket, TicketTrack>(tracks => tracks
            .When(TicketTrack.Refund, t => t.Chain<IssueRefund>().Chain<NotifyCustomer>(),
                  requireConfidence: 0.7)
            .When(TicketTrack.Escalate, t => t.Chain<OpenIncident>())
            .When(TicketTrack.SelfServe, t => t.Chain<SendHelpArticle>())
            .RequireConfidence(0.5)
            .Otherwise(t => t.Chain<QueueForHuman>()))
        .Chain<CloseTicket>()
        .Resolve();
```

The decider can be a typed decision model (one state and typed questions in, calibrated
probabilities out, no generated text), a rule table, a larger language model, or a test double.
The train does not know which. `Trax.Docs/adr/0040` records why a chain branches this way and no
other.

## Questions and answers

Every decision is one of three kinds of question, each with its own routing step.

| Question | Declared with | Answer | Routed by | Memory holds |
|---|---|---|---|---|
| Which of these options? | `Choice<TTrack>()`, an enum | the chosen member, a confidence, each member's probability | [Switch](/docs/sdk-reference/train-methods/switch) | `ChoiceDecision<TTrack>` |
| Where on this ordered scale? | `Score<TLevel>()`, an enum whose values run lowest to highest | a position from 0 to the top level, a confidence, each level's probability | [Scale](/docs/sdk-reference/train-methods/scale) | `ScoreDecision<TLevel>` |
| How likely is yes? | `YesNo<TQuestion>()`, a marker type | the probability of yes | [Gate](/docs/sdk-reference/train-methods/gate) | `YesNoDecision<TQuestion>` |

A model sees the question's words and each option's description, never the type's name. The
words go on the type, once:

```csharp
[Asks("Which team should handle this support ticket?")]
public enum TicketTrack
{
    [Description("The customer wants their money back for an order.")]
    Refund,

    [Description("An outage, a security problem or a legal threat. Page whoever is on call.")]
    Escalate,

    [Description("A how-to question the help centre already answers.")]
    SelfServe,
}

[Asks("Does this post threaten violence against a person?",
      Yes = "It threatens, or incites, harm to someone.",
      No = "It does not threaten anyone, even if it is rude.")]
public sealed class ContainsThreat;
```

An `asking:` argument where the decision is declared overrides `[Asks]`. A decision with no
question at all is refused when the chain is read.

## Asking inline, or asking first

A routing step can ask its own question:

```csharp
.Switch<LoanApplication, Underwriting>(tracks => tracks ...)   // asks, then routes
```

Or a [Decide](/docs/sdk-reference/train-methods/decide) step can ask several questions about one
state in a single call, and later steps route on the answers without asking again. That is one
round trip to a model instead of three:

```csharp
AddServices(decider)
    .Decide<Post>(q => q.YesNo<ContainsThreat>().Choice<Verdict>().Score<Severity>())
    .Gate<ContainsThreat>(gate => gate
        .Yes(t => t.Chain<TakeDownPost>(), atLeast: 0.7)
        .No(t => t.Switch<Verdict>(verdict => verdict
            .When(Verdict.Allow, a => a.Chain<PublishPost>())
            .When(Verdict.Remove, r => r.Chain<TakeDownPost>())
            .When(Verdict.Review, r => r.Scale<Severity>(scale => scale
                .AtLeast(Severity.Low, s => s.Chain<QueueForModerator>())
                .AtLeast(Severity.High, s => s.Chain<PageTrustAndSafety>())))),
            below: 0.3)
        .Unsure(t => t.Chain<PageTrustAndSafety>()))
    .Resolve();
```

Routing steps nest: a track is a chain like any other, so it can ask and route again. Write each
track on the parameter it is handed (`t => t.Chain<X>()`), not on the train's own chain methods.

## When the answer is not followed

Three things decide whether the train takes the track the decider chose.

| Situation | What happens |
|---|---|
| The choice's confidence is below its track's `requireConfidence`, or below the switch's `RequireConfidence` when the track sets none | The `Otherwise` track runs. Without one, the run fails. |
| The decider chose a member the switch has no track for | The `Otherwise` track runs. Without one, the run fails. |
| A gate's probability falls between its No and Yes bars | The `Unsure` track runs. Without one, the run fails. |

Set the bars per track. The track with the gravest consequence gets the highest bar and the
safest gets the lowest, often none: in underwriting, `Approve` and `Decline` might need 0.95
while `ManualReview` accepts anything. A switch that declares no `Otherwise` refuses to guess,
which is what a train whose outcomes all have consequences wants.

A choice's confidence depends on how many options it was offered, so adding a track moves every
threshold on that switch. Re-check them against labelled cases after changing a switch's tracks,
its question's words, or the model's version.

The track taken goes in Memory as `TrackTaken<TKey>`, with the reason when the decision was not
followed, so a later junction can record or act on it.

## When the run fails instead

An answer Trax cannot trust is never acted on. The run fails, the failure names the step
(`Switch<Ticket, TicketTrack>`), and it is
[classified](/docs/core/trains-and-junctions#classifying-failures) `Permanent`, because asking the
same declaration again gets the same answer:

- an option that is not a member of the enum, including a member given as a number
- a confidence or probability outside 0 to 1, or not a number
- a score below 0 or above the top level
- the wrong kind of answer, or no answer to a question that was asked
- a declaration the startup check refuses, for a host that skips the check

A decider that throws fails the run with its own exception, whatever fallback tracks are
declared: an error is not a decision. The decider's own failure class is kept, so an adapter can
mark a throttled model `Transient`.

## Deciders

`IDecider` takes a `DecisionRequest` (the train's name, the state, the questions) and returns one
answer per question. It is found the way a junction input is: in Memory, then the container. One
registration serves every decision in every train; `Decide(q => q.DecidedBy<TDecider>())` names a
different one for a single step.

| Decider | Package | For |
|---|---|---|
| `SystemOneDecider`, from `AddNimbleDecider` | Trax.Effect.Decisions.SystemOne | Nimble, Bespoke Labs' open-weights decision model, on a local Ollama or hosted. The default choice. See [Nimble](/docs/effect/decisions#nimble). |
| `SystemOneDecider`, from `AddSystemOneDecider` | Trax.Effect.Decisions.SystemOne | Any other model that speaks the System One request format: Jev, d1, Laya, Kev and others. |
| `RuleDecider` | Trax.Core | Policy rather than judgement, written as code. Always certain. |
| `CascadingDecider` | Trax.Core | A fast decider first, and a slower one only for the questions the first was unsure of. |
| `ScriptedDecider` | Trax.Core | Answers written in advance, for tests and for running locally before a model is wired up. |

```csharp
var policy = new RuleDecider().Choice<LoanApplication, Underwriting>(application =>
    application switch
    {
        { CreditScore: >= 740, DebtToIncome: <= 0.36m } => Underwriting.Approve,
        { CreditScore: < 580 } or { DebtToIncome: > 0.5m } => Underwriting.Decline,
        _ => Underwriting.ManualReview,
    });
```

### Escalating what the fast decider is unsure of

```csharp
services.AddSingleton<IDecider>(sp => new CascadingDecider(
    first: sp.GetRequiredService<SystemOneDecider>(),   // Nimble
    then: sp.GetRequiredService<LargeModelDecider>(),
    escalateBelow: 0.8));
```

`LargeModelDecider` stands for an `IDecider` of your own over a chat model; Trax does not ship
one. A choice or score below `escalateBelow`, a yes/no whose probability lies strictly between
`unsureAbove` (0.2) and `unsureBelow` (0.8), and a question the first decider did not answer are
asked again of the second. Only those questions are sent. The second answer replaces the first;
if the second decider fails, the run fails. The switch's own bars and fallback tracks still
apply to whatever the cascade returns, which is how a person becomes the third tier.

### Comparing a new decider without trusting it

```csharp
.Decide<Post>(q => q.YesNo<ContainsThreat>().Choice<Verdict>().Shadow<ICandidateDecider>())
```

A shadow is asked every question alongside the live decider. Its answers are recorded with
whether each reached the same outcome, and never acted on; a shadow that fails or disagrees
changes nothing. Use it to move from a rule table to a model, or from one model version to the
next, on real traffic before switching over. A shadow handed to `AddServices` is passed as an
interface, as every service there is; from the container, a concrete type does.

## What the startup check verifies

Every track is read when the chain is declared, and each is replayed from the Memory at the
routing step. After a routing step the chain can rely only on what **every** track produces:
a junction after a switch that needs a value only one track makes is refused, naming it. A fault
inside a track is reported at the routing step, as `track 'Refund', step 1: ...`.

The check also refuses a routing step with no decision before it, a decider or shadow that
neither Memory nor the container supplies, a track declared twice, a switch with no tracks, a
gate with no Yes or No track or with bars that cross, and a scale with no track for its lowest
level. Reading the chain asks no decider and runs no track.

## Observing and replaying

Core reports every answered question and every routing to an `IDecisionObserver`, and asks an
`IDecisionReplay` for an earlier answer before it asks a decider. Both are optional and found in
Memory, then the container; an observer that throws is ignored. Trax.Effect implements both, to
record decisions against the run and to make a re-queued run take the tracks the original took.
See [Decision Recording and Models](/docs/effect/decisions).

## Testing a train's decisions

```csharp
var decider = new ScriptedDecider().Choose(Underwriting.Approve, confidence: 0.62);

var result = await new UnderwriteLoan(decider).RunEither(new LoanApplication("a3", 700, 0.4m));

result.IsLeft.Should().BeTrue();          // below Approve's 0.95, and no Otherwise
decider.Requests.Should().ContainSingle(); // what a model would have been shown
```

`ScriptedDecider` answers `Choose`, `Score` and `YesNo` with what the test says, can answer a
question with an answer that does not fit (`Answer(key, ...)`) or fail every request (`Throws`),
and keeps every request it was asked. A question it has no answer for is left unanswered, which
fails the run the way a model that skipped one would.

## SDK Reference

> [Decide](/docs/sdk-reference/train-methods/decide) | [Switch](/docs/sdk-reference/train-methods/switch) | [Gate](/docs/sdk-reference/train-methods/gate) | [Scale](/docs/sdk-reference/train-methods/scale) | [AddServices](/docs/sdk-reference/train-methods/add-services) | [DeclaredChain](/docs/sdk-reference/train-methods/declared-chain) | [AddNimbleDecider](/docs/sdk-reference/configuration/add-nimble-decider)
