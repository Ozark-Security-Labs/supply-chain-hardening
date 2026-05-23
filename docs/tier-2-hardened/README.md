# Tier 2 — Hardened

> Status: outline only. Patterns ship in **v0.2.0** (see [project status](../../README.md#status)).

Tier 2 turns the Tier 1 conventions into enforced rules and adds the dependency-aware CI checks that catch regressions in PRs. You're no longer relying on individual maintainer discipline.

Worked example throughout this tier: [`Ozark-Security-Labs/forkguard`](https://github.com/Ozark-Security-Labs/forkguard).

## Planned patterns

1. **Wire `deterministic-deps`** — advisory mode first, then enforce mode at `severity-threshold: low`
2. **Hash-pinned Python requirements** — `uv pip compile --generate-hashes` or `pip-compile --generate-hashes`
3. **Container image digests** — `name:tag@sha256:<digest>` in Dockerfile, devcontainer, compose
4. **Dependency Review on PRs** — block PRs that introduce vulnerable or license-incompatible deps
5. **Dependabot or Renovate with grouped updates** — keep pins fresh without PR fatigue
6. **`persist-credentials: false` and harden-runner** — limit blast radius of a compromised step
7. **CodeQL on push + PR** — catch first-party security regressions in the same code-scanning surface

## Prerequisites

You should be cleanly on [Tier 1 — Baseline](../tier-1-baseline/) before adopting Tier 2.

## Next tier

When you're ready to make your releases independently verifiable, move on to [Tier 3 — Production](../tier-3-production/).
