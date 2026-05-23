# Tier 1 — Baseline

Tier 1 is the floor every repo should clear on day one. It costs almost nothing and delivers the largest single jump in supply-chain integrity: known dependencies, known build inputs, and CI tokens scoped to what each job actually needs.

The patterns in this tier remove ambiguity. After Tier 1, every install reproduces the same dependency tree byte-for-byte, every CI run executes the same action source code, and every job has only the GitHub token scopes it actually needs.

## Patterns

1. [Commit a lockfile](./commit-a-lockfile.md) — every package manager, frozen in CI
2. SHA-pin every third-party GitHub Action _(coming in a later v0.1.x)_
3. Pin the runner OS _(coming in a later v0.1.x)_
4. Pin the language toolchain _(coming in a later v0.1.x)_
5. Minimal workflow permissions _(coming in a later v0.1.x)_

## Cross-tier worked example

[`Ozark-Security-Labs/deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) is the most compact public OSL repo that demonstrates the Tier 1 patterns in context. Each pattern's "Real example" link points at the specific file in that repo (or another OSL repo where the pattern is shown more cleanly for a given ecosystem).

## When you're ready

When the patterns above are routine for you, move on to [Tier 2 — Hardened](../tier-2-hardened/) to add active CI enforcement on top of Tier 1's discipline.
