---
layout: default
title: Supply Chain Security
description: "The controls protecting every Trax package from source to nuget.org: pinned dependencies, split release jobs, keyless publishing, provenance and gated releases."
nav_order: 11
section: Guides
---

# Supply Chain Security

Trax is built and published through a defense-in-depth pipeline. No single control is load-bearing: each layer bounds what a compromise of any other layer can reach, so a poisoned dependency, a hijacked action, or a tampered runner yields little on its own. This page describes the controls that protect every Trax package from source to nuget.org.

The working assumption throughout is that any code in a CI job may be compromised and any externally-influenced input may be hostile. The controls keep untrusted code and input away from credentials, bound what a compromised component reaches, stop stolen data from leaving, and keep every credential short-lived.

## Pinned, integrity-verified dependencies

Every repository uses Central Package Management. Cross-repo `Trax.*` references and third-party packages are pinned in a single `Directory.Packages.props` per repo, and individual project files carry no inline versions. The pins are exact versions, with one recorded exception: Trax.Dashboard lets `Radzen.Blazor` and its test-only `Microsoft.Extensions.DependencyInjection` float within their major (`11.*`, `10.*`), and the committed lockfile still fixes the resolved version and its hash. Each project commits a `packages.lock.json` capturing the full transitive graph with content hashes.

CI restores with `dotnet restore --locked-mode`, which fails the build on any drift from the committed lockfile, and with the npm release tooling locked the same way (`npm ci --ignore-scripts`). Install scripts are disabled, so a package cannot execute code merely by being restored. A poisoned or substituted transitive dependency cannot enter a build unnoticed, and builds resolve the same bytes every time.

| Control | Mechanism |
|---|---|
| Version pinning | Central Package Management (`Directory.Packages.props`) |
| Transitive integrity | committed `packages.lock.json` + `dotnet restore --locked-mode` |
| No install-time execution | `npm ci --ignore-scripts` for release tooling |
| Lockfile hygiene | CI guard rejects a lockfile containing a local-only resolved version |

## Separated build, release, and publish

Each release pipeline is split into three jobs, because a job is the isolation unit (a fresh runner with its own secret access):

- **build / test** restores, compiles and runs the tests, and holds no release or publish credential.
- **release** runs semantic-release with a token that can write tags and GitHub releases. When a version is cut, the same job restores the locked dependencies again and runs `dotnet pack`, so it builds the package artifact. It does not hold the publish credential.
- **publish** downloads that artifact, attests it and pushes it unchanged. It runs no restore, build or test code and holds the only credential that can publish.

The publish credential is therefore never present in a job that executes dependency code. The release job is not that clean: restoring and packing runs dependency code (MSBuild targets, analyzers, source generators) next to a `contents: write` token, and that code shapes the `.nupkg` the publish job attests. Moving the pack into the build job, so the release job only tags and hands the artifact on, is planned and not done.

## Least-privilege workflow tokens

Workflows default to `permissions: {}` and widen only what each job proves it needs. The build job is `contents: read`. Checkout runs with `persist-credentials: false`, so a token is never left in `.git/config` for later steps to read, in every job but one: the Dependabot lockfile heal's commit job keeps its checkout credentials to fetch the base commit, and makes its commit through the GitHub API rather than with `git push`. That job runs no dependency code; the lockfile patch it applies comes from a separate job that does, and is restricted to lockfile paths. Secrets are bound to the single step that uses them and referenced as quoted shell variables, never interpolated into a command line.

## Pinned actions and images

Every `uses:` reference is pinned to a full commit SHA, and every service-container image is pinned to a digest. A maintainer (or an attacker who moves a tag) cannot silently change what a workflow executes on its next run. Updates arrive as reviewable changes to the pinned SHA.

## Keyless publishing

Packages are published to nuget.org with Trusted Publishing. The publish job exchanges a short-lived GitHub OIDC token for a temporary, single-use nuget.org API key (valid roughly one hour) scoped by a publishing policy to the exact repository, workflow file, and deployment environment. No long-lived publishing key is stored, rotated, or exposed.

## Signed build provenance

Every published package carries SLSA build provenance, generated keylessly in the publish job and recorded in the Sigstore transparency log. The attestation binds each package digest to the repository, workflow, and commit that produced it. Consumers can verify it:

```bash
gh attestation verify <package>.nupkg --repo TraxSharp/<repo>
```

## Deterministic builds

Builds are deterministic: the same source and toolchain produce byte-for-byte identical output. Module identifiers and timestamps are content-based rather than random, and CI normalizes embedded source paths. Determinism is the prerequisite for independent verification that a published binary matches its source.

## Gated releases

A merge to `main` builds and tests but releases nothing; a release is a manual run of the release workflow. Both the release and publish jobs run behind the protected `release` deployment environment, which restricts which refs may deploy and can require a human reviewer before a version is cut or a package leaves the pipeline. The environment also scopes the OIDC claim that the trusted-publishing policy checks, a claim a feature branch or pull request cannot forge.

## Runtime egress monitoring

Every job runs under StepSecurity Harden-Runner, in `egress-policy: audit` mode. It records each job's outbound connections, which establishes the baseline an allowlist would be built from, but it does not block anything: code that reads a credential could still send it to an arbitrary host. Switching to `block` with a tuned allowlist is planned and not done.

## Untrusted input handling

Externally-influenced fields (pull request titles, branch names, commit messages) are never interpolated into a shell or script context. Workflows run pull request code only in the restricted `pull_request` context, never in a privileged context with secrets. This keeps both untrusted code and untrusted data away from anything that could turn them into execution.

## Enforced conventions

The conventions above are enforced, not just documented. `Trax.Core.Testing` ships architecture guards for the dependency model (centrally-managed versions, no inline versions on cross-repo references), repository structure, and test hygiene. Trax.Core, Trax.Effect, Trax.Mediator and Trax.Dashboard run them against their own tree. Trax.Scheduler, Trax.Api, Trax.Cli and Trax.Samples enforce their conventions with their own `Tests.Meta` suites instead. Either way, a change that violates a checked convention fails CI rather than drifting in silently.

## Layered, not trusted

The goal is not a set of components assumed to be trustworthy, since trust is what gets exploited. It is an arrangement where compromising any one component yields little, is detectable, and is recoverable. SHA pinning bounds what a compromised action becomes; job separation keeps the publish credential away from dependency code; egress monitoring records where data goes, and once it blocks will bound it; keyless, short-lived credentials bound how long a stolen one lives; provenance and determinism let the result be verified from outside the pipeline.
