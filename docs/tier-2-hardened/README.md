# Tier 2 — Hardened

Tier 2 makes the Tier 1 disciplines _enforced_ rather than aspirational. After Tier 1, your repo _can_ be deterministic — lockfiles committed, actions SHA-pinned, runners and toolchain pinned. Nothing in Tier 1 fails when a regression slips in. Tier 2 adds the CI checks that block PRs introducing non-determinism, unvetted dependencies, or vulnerable first-party code; the bot-driven update cadence that keeps pinned versions from going stale; and the runner-level mitigations that bound the blast radius of a compromised step.

You should be cleanly on [Tier 1 — Baseline](../tier-1-baseline/) before adopting Tier 2.

## Patterns

1. [Wire `deterministic-deps` into CI](./wire-deterministic-deps.md) — advisory mode first, then enforce at the lowest severity threshold
2. [Hash-pin every Python requirement](./hash-pin-python.md) — `--hash=sha256:` for every requirement; a lockfile alone isn't enough on PyPI
3. [Pin container image digests](./container-image-digests.md) — Dockerfile, Compose, and devcontainer references must use `@sha256:<digest>`
4. [Run Dependency Review on every PR](./dependency-review.md) — block PRs that introduce vulnerable, malicious, or license-incompatible deps
5. [Use Dependabot or Renovate with grouped updates](./grouped-dependency-updates.md) — keep pins fresh without PR fatigue
6. [`persist-credentials: false` and harden-runner](./harden-runner-and-credentials.md) — strip the token from git config and audit (then block) runner egress
7. [CodeQL on push and PR](./codeql.md) — catch first-party security regressions alongside dep findings

## Worked examples

[`forkguard`](https://github.com/Ozark-Security-Labs/forkguard) demonstrates most of these patterns in their hardened-but-still-readable form: `deterministic-deps` in enforce mode (Rule 2.1), Dependency Review (Rule 2.4), CodeQL (Rule 2.7), and Dependabot wiring (Rule 2.5, ungrouped). [`SessionScope`](https://github.com/Ozark-Security-Labs/SessionScope) adds the `dependency-review` event-gating pattern and an audit retry-with-cache for `cargo audit`. Each pattern's "Real example" link points at the specific file demonstrating it.

Three patterns currently lack an in-tree OSL real example and link to canonical external references instead:

- **Rule 2.2 (hash-pinned Python)** — no OSL Python project ships hashed `requirements.txt` today
- **Rule 2.3 (container image digests)** — no OSL project ships a Dockerfile today
- **Rule 2.6 (harden-runner)** — no OSL workflow uses it today; `persist-credentials: false` is demonstrated by PkgWarden's Scorecard workflow

Closing those gaps is a near-term follow-up.

## When you're ready

Move on to [Tier 3 — Production](../tier-3-production/) to make your releases independently verifiable.
