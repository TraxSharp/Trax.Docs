---
authors: [Theauxm]
repos: [core, effect, mediator, scheduler, api, dashboard, cli, samples]
areas: [platform, testing]
status: accepted
---

# A junction chain is a declaration, not a step of the work

`Junctions()` names the junctions a train passes through and nothing else. The accessors for
per-execution state (`TrainInput`, `TrainOutput`) throw while a chain is being declared, so a
chain cannot vary by the value being processed. That makes every train's chain readable at host
startup, from types alone, before any of the work it describes has run.

## Status

**Accepted.**

## Why this is written down

Because the restriction looks arbitrary until you try to check a chain that does not obey it.

A chain that branches on its input does not have one shape, it has several. Reading it yields
whichever shape the reading conditions happen to select, so the chain verified at startup need
not be the chain that runs, and a green startup check would mean nothing. The value of eager
verification depends entirely on there being a single chain to verify.

The rule was also already true everywhere it mattered and simply unenforced. All 96 `Junctions()`
overrides across the eight repos were already pure when the gate was added; the only violations
found anywhere were in a consumer's generated `POST` trains, which read the input to stamp an
identifier before building the chain.

## Considered options

**A static chain declaration.** A `static abstract` interface member has no `this`, so
per-execution state would be unreachable by the compiler rather than at runtime, and branching on
ambient state such as the clock would be closed off too. Rejected for now on cost: it changes the
shape of every train in every repo, and it cannot carry the `AddServices<T>(T service)` and
`Chain<TJunction>(TJunction instance)` overloads, which take values rather than types. The
runtime gate gets the same guarantee for input-dependence at zero migration. This remains the
option to take if ambient branching ever turns up in practice.

**A Roslyn analyzer.** `TrainChainAnalyzer` already parses these chains, so it could report a
chain that reads the input. Rejected as the primary mechanism because an analyzer has to be
referenced to run, which makes correctness opt-in and silently absent wherever someone forgot.
An analyzer remains useful as a second, earlier signal; it is not what the guarantee rests on.

**Reading the chain from IL.** Inspecting the method body avoids both executing the chain and
changing its shape. Rejected as fragile across compiler versions and effectively undebuggable
when it reads a body wrongly.

## Consequences

**Work that needs the input moves into a junction.** That is where per-execution decisions
belong, and a junction receives the input as its argument. A train that wants to stamp an
identifier onto its input declares a junction that does it.

**Reading a chain resolves nothing.** Steps are recorded from type arguments, so a junction that
cannot be constructed is still declared. Whether a junction resolves is a separate check against
the container, which keeps the two failures distinguishable.

**Branching on ambient state is still possible.** A chain that reads the clock or a static flag
records whichever shape boot-time conditions select, with no signal that it did. Instance state
is closed by construction; ambient state is merely unlikely. Closing it needs the static
declaration above.

**`RunInternal` and `Activate` are no longer reachable.** A train that built its chain
imperatively could not be read, so the escape hatch and the guarantee could not both exist.
`RunInternal` is private and `Activate` is internal, which means `Junctions()` is the only way to
declare a chain and every train is therefore readable. Closing them cost four `Chain` overloads
and a parameterless `Resolve()`, added so the declaration can express what the imperative form
could.

**Seeding arbitrary values into Memory is gone with it.** `Activate(input, otherInputs)` had no
declarative equivalent, and adding one would have let a declaration inject values the junctions
it names never produced. The trains that used it were restructured: a value a later junction
reads is now produced by an earlier junction, which is where work belongs.

## Exemplars

**Enforced elsewhere:** `DeclaredChainTests` in Trax.Core pins that every step is recorded in
order with its junction's input and output types, that no junction runs, that an unconstructible
junction is still declared, and that the train is runnable afterwards. `ChainDeclarationTests` in
Trax.Effect pins that reading `TrainInput` or `TrainOutput` while declaring throws, naming the
train and the member, and that a chain naming only junctions declares cleanly.

Not covered: nothing detects a chain that branches on ambient state, and nothing yet fails a host
whose train reads its input. The gate throws where it is called; wiring it into startup
verification is the follow-on work this decision exists to enable.

## Changelog

- **2026-09-22**: `RunInternal` made private and `Activate` internal, once every train in the
  workspace declared its chain through `Junctions()`.
- **2026-09-22**: Recorded.
