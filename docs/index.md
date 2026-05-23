# Supply Chain Hardening

> Status: scaffold (**v0.0.1**) — content lands in `v0.1.0` and beyond.

This site mirrors <https://github.com/Ozark-Security-Labs/supply-chain-hardening>, where the canonical source lives. Edits and PRs happen on GitHub; the rendered site here updates when a new release tag is cut.

## Pick your tier

- [**Tier 1 — Baseline**](./tier-1-baseline/) — every repo, day one. Lockfile, SHA-pinned actions, pinned runner, pinned toolchain, minimal CI permissions. Worked example: [`osl-glob`](https://github.com/Ozark-Security-Labs/osl-glob).
- [**Tier 2 — Hardened**](./tier-2-hardened/) — active enforcement. `deterministic-deps`, hash-pinned Python, container digests, dependency review, CodeQL, harden-runner. Worked example: [`forkguard`](https://github.com/Ozark-Security-Labs/forkguard).
- [**Tier 3 — Production**](./tier-3-production/) — verifiable releases. SLSA Build L3, SBOM, signed artifacts, OIDC, Scorecard, branch + tag protection. Worked example: [`SessionScope`](https://github.com/Ozark-Security-Labs/SessionScope).

## Appendices

See [the appendix index](./appendix/) for tool comparisons, fork hygiene, migration playbooks, and a reviewer's rubric.
