# Supply Chain Hardening

Recipe-driven best practices for deterministic dependency declarations and reducing supply chain attack surface, from baseline (every repo, day one) to production-grade verifiable releases.

[![Deterministic Deps](https://github.com/Ozark-Security-Labs/supply-chain-hardening/actions/workflows/deterministic-deps.yml/badge.svg)](https://github.com/Ozark-Security-Labs/supply-chain-hardening/actions/workflows/deterministic-deps.yml)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/Ozark-Security-Labs/supply-chain-hardening/badge)](https://scorecard.dev/viewer/?uri=github.com/Ozark-Security-Labs/supply-chain-hardening)
[![Prose: CC BY 4.0](https://img.shields.io/badge/Prose-CC%20BY%204.0-lightgrey.svg)](LICENSE)
[![Examples: MIT](https://img.shields.io/badge/Examples-MIT-blue.svg)](LICENSE-CODE)

> Rendered at <https://docs.ozarksecuritylabs.com/supply-chain/>.
> Source of truth lives here on GitHub; PRs welcome.

---

## What this is

A working maintainer's guide to making your software supply chain harder to attack. The patterns here are organized into three tiers of increasing rigor. Pick the tier that fits where your project is today, and adopt the next one when you have the appetite.

## What this isn't

- A theoretical threat-modeling treatise. Every pattern is something you can copy-paste into a real repo today.
- A vendor pitch. [`deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) — an action published by the same org that maintains this guide — is recommended as one enforcement tool, but every pattern is presented with its rule and verification independent of any tool. The [Tool Landscape](docs/appendix/tool-landscape.md) appendix names the alternatives honestly.
- An exhaustive catalogue. We cover the patterns that demonstrably reduce attack surface for the majority of OSS projects. Esoteric or single-ecosystem niche techniques are out of scope.

## Pick your tier

| Tier | Goal | Worked example | Status |
| --- | --- | --- | --- |
| **[Tier 1 — Baseline](docs/tier-1-baseline/)** | Every repo, day one: lockfile, SHA-pinned actions, pinned runner, pinned toolchain, minimal CI permissions | [`osl-glob`](https://github.com/Ozark-Security-Labs/osl-glob) | _Coming in v0.1.0_ |
| **[Tier 2 — Hardened](docs/tier-2-hardened/)** | Active enforcement: `deterministic-deps`, hash-pinned Python, container digests, dependency review, CodeQL, harden-runner | [`forkguard`](https://github.com/Ozark-Security-Labs/forkguard) | _Coming in v0.2.0_ |
| **[Tier 3 — Production](docs/tier-3-production/)** | Verifiable releases: SLSA Build L3, SBOM, signed artifacts, OIDC, Scorecard, branch + tag protection | [`SessionScope`](https://github.com/Ozark-Security-Labs/SessionScope) | _Coming in v0.3.0_ |

Each tier is self-contained — adopting Tier 1 in full delivers measurable reduction in attack surface even if you never touch Tier 2.

## How each pattern is structured

Every pattern in Tiers 1–3 answers the same questions in the same order, so you can scan or deep-read:

1. **Rule** — one sentence.
2. **Why it matters** — the attack it prevents, with a CVE or incident link where one exists.
3. **How to do it** — copy-pasteable config, tabbed where ecosystems differ.
4. **How to verify** — the `deterministic-deps` rule ID, alternative tools, and a manual one-liner for the no-CI case.
5. **Common pitfalls** — the 1–3 gotchas that bite people.
6. **Real example** — a commit-SHA permalink to a file in a public OSL repo demonstrating it.

## Appendices

- **[A. Tool landscape](docs/appendix/tool-landscape.md)** — `deterministic-deps` vs `pin-github-action`, StepSecurity, Renovate SHA mode, Scorecard's pinned-dependencies check
- **[B. Fork hygiene](docs/appendix/fork-hygiene.md)** — when and how to fork-and-trim a transitive dependency
- **[C. Migration playbook](docs/appendix/migration-playbook.md)** — introducing these patterns to a live repo without nuking dev velocity
- **[D. Reviewer's rubric](docs/appendix/reviewers-rubric.md)** — one-page checklist for procurement and audit readers
- **[E. Verification cookbook](docs/appendix/verification-cookbook.md)** — what to grep for, what artifacts to expect

## Status

| Release | Contents |
| --- | --- |
| `v0.0.1` | Repo scaffold, dogfooded CI, license, contribution model — proves the publishing pipeline before any content lands |
| `v0.1.0` | Introduction + threat model + all Tier 1 patterns |
| `v0.2.0` | All Tier 2 patterns + hash-pinned tooling configs |
| `v0.3.0` | All Tier 3 patterns + SLSA + SBOM + signed releases (this repo starts emitting them too) |
| `v1.0.0` | All appendices + external reviewer pass complete |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the per-pattern entry template and the PR workflow. Issues are welcome for proposing new patterns, surfacing alternative tools, or flagging when a "Real example" link rots.

## License

| Scope | License |
| --- | --- |
| Prose under `/docs` and this README | [CC BY 4.0](LICENSE) |
| Code, config, and YAML examples under `/examples` and `.github/` | [MIT](LICENSE-CODE) |

When in doubt about which license governs a given file, the prose license applies.

## Related work

- [`deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) — the GitHub Action that enforces the determinism rules cited throughout this guide
- [OpenSSF Scorecard](https://scorecard.dev/) — automated assessment of OSS project security health
- [SLSA](https://slsa.dev/) — Supply-chain Levels for Software Artifacts
- [Sigstore](https://www.sigstore.dev/) — keyless artifact signing and transparency log
