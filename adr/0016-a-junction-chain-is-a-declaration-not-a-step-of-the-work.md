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

The rule was also already true everywhere it mattered and simply unenforced. None of the 96
`Junctions()` overrides across the eight repos read the input when the gate was added; the only
violations found anywhere were in a consumer's generated `POST` trains, which read the input to
stamp an identifier before building the chain.

## Considered options

**A static chain declaration.** A `static abstract` interface member has no `this`, so
per-execution state would be unreachable by the compiler rather than at runtime, and branching on
ambient state such as the clock would be closed off too. Rejected for now on cost: it changes the
shape of every train in every repo, and it cannot carry the `AddServices<T>(T service)` and
`Chain<TJunction>(TJunction instance)` overloads, which take values rather than types. The
runtime gate gets the same guarantee for input-dependence at zero migration. This remains the
option to take if ambient branching ever turns up in practice.

**A Roslyn analyzer.** `TrainChainAnalyzer` parses fluent chains, so it could in principle report
a chain that reads the input. It only inspects chains rooted at `Activate()`, though, and with
`Activate` internal it no longer sees any chain a train can write; it is deprecated. Rejected as
the primary mechanism because an analyzer has to be referenced to run, which makes correctness
opt-in and silently absent wherever someone forgot.
An analyzer remains useful as a second, earlier signal; it is not what the guarantee rests on.

**Reading the chain from IL.** Inspecting the method body avoids both executing the chain and
changing its shape. Rejected as fragile across compiler versions and effectively undebuggable
when it reads a body wrongly.

## Consequences

**Work that needs the input moves into a junction.** That is where per-execution decisions
belong, and a junction receives the input as its argument. A train that wants to stamp an
identifier onto its input declares a junction that does it.

**Reading a chain resolves nothing.** Steps are recorded from type arguments, so a junction that
cannot be constructed is still declared. Whether a junction can be constructed is not checked at
all: a junction takes its constructor arguments from Memory, which the chain fills as it runs, so
the answer is not decidable from the declaration. The container is consulted only to decide
whether a junction's input can be supplied from outside Memory.

**A declaration that does work is refused.** `Junctions()` runs when a chain is read, so a body
that awaits before returning, returns a result instead of ending in `Resolve()`, or ends in
`Resolve(value)` has no chain to read. Each is recorded as a refusal and the host will not
start, instead of the train reading as an empty, clean chain. Whatever such a body started at
boot is not undone.

**The replay has to mirror the runtime.** A chain is only as verified as the replay is faithful.
The train's input and tuple elements enter Memory under their type and interfaces, but a
junction's output, an extracted value and an `AddServices` value enter under exactly one type,
and lookup is by exact type before the container. A short circuit's output is not available to
what follows it, because the path that continues is the one where it returned Left.

**Branching on ambient state is still possible.** A chain that reads the clock or a static flag
records whichever shape boot-time conditions select, with no signal that it did. Only
`TrainInput` and `TrainOutput` are gated; branching on `Metadata`, an injected property or any
other instance state is equally unchecked. Closing that needs the static declaration above.

**`RunInternal` and `Activate` are no longer reachable.** A train that built its chain
imperatively could not be read, so the escape hatch and the guarantee could not both exist.
`RunInternal` is private and `Activate` is internal, which means `Junctions()` is the only way to
declare a chain and every train is therefore readable. Closing them cost four `Chain` overloads
and a parameterless `Resolve()`, added so the declaration can express what the imperative form
could.

**Seeding extra inputs is gone with it.** `Activate(input, otherInputs)` had no declarative
equivalent, and the trains that used it were restructured so that a value a later junction reads
is produced by an earlier junction. `AddServices(value)` and `Extract(value)` still put a value in
Memory directly; the replay records each as a seed of its declared type.

## Exemplars

**Enforced elsewhere:** `DeclaredChainTests` in Trax.Core pins that every step is recorded in
order with its junction's input and output types, that no junction runs, that an unconstructible
junction is still declared, and that the train is runnable afterwards. `ChainDeclarationTests` in
Trax.Effect pins that reading `TrainInput` or `TrainOutput` while declaring throws, naming the
train and the member, and that a chain naming only junctions declares cleanly.
`DeclaredChainTests` also pins the refusals: an awaiting body, a direct result, `Resolve(value)`,
and an async body's exception being rethrown. `TrainChainStartupValidatorTests` in Trax.Mediator
pins that a host refuses to start when a train names a junction whose input never reaches Memory,
declares a chain that reads the input (synchronously or in an async body), or does work before
declaring; that it reports every failing train at once; that a container-supplied input is
checked without building the service; and that the opt-out works. `ChainVerificationTests` in
Trax.Core pins the replay itself: a junction output's interfaces are not available and the run
fails the same way, a short circuit neither supplies the return value nor its output, seeds and
tuples are available, `Extract` ignores the container and `IChain` needs its junction.

Not covered: nothing detects a chain that branches on ambient state or on instance state other
than the input and output. The replay knows the train's declared input type, not the subtype that
flows, so a junction asking for an interface only that subtype implements reads as a fault; that
is why the check has an opt-out rather than being unconditional.

## Changelog

- **2026-09-23**: The replay was reading a junction output's interfaces as available and letting
  a short circuit excuse the rest of the chain, both of which the runtime does not do, so it passed
  chains that failed on every run. It now mirrors the runtime, and a declaration that awaits,
  returns a result or ends in `Resolve(value)` is refused. Corrected the overstatements about
  instance state being closed and value seeding being gone.
- **2026-09-23**: Corrected two claims. The considered-options entry said `TrainChainAnalyzer`
  already parses these chains; it only parses chains rooted at `Activate()`, which no train can
  write any more, so it checks nothing and is deprecated. The consequences entry said whether a
  junction resolves is a separate check against the container; that check was removed
  (Trax.Mediator `a20ce54`) because constructibility is not decidable from the declaration. The
  decision is unchanged.
- **2026-09-22**: Wired into startup: a host now replays every registered train's chain and
  refuses to start when one cannot run.
- **2026-09-22**: `RunInternal` made private and `Activate` internal, once every train in the
  workspace declared its chain through `Junctions()`.
- **2026-09-22**: Recorded.
