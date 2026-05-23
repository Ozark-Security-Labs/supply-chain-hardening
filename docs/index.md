# Supply Chain Hardening

Recipe-driven best practices for deterministic dependency declarations and reducing supply chain attack surface, from baseline (every repo, day one) to production-grade verifiable releases.

## What this is

A working maintainer's guide to making your software supply chain harder to attack. Three tiers of increasing rigor. Pick the tier that fits your project today; adopt the next one when you have the appetite.

## What this isn't

- A theoretical threat-modeling treatise. Every pattern is something you can copy-paste into a real repo today.
- A vendor pitch. [`deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) — an action published by the same org that maintains this guide — is recommended as one enforcement tool. Every pattern is presented with its rule and verification independent of any single tool, and the [Tool Landscape](./appendix/tool-landscape.md) appendix names alternatives honestly.
- An exhaustive catalogue. We cover patterns that demonstrably reduce attack surface for the majority of OSS projects. Niche or single-ecosystem techniques are out of scope.

## The threat model

Six attack classes that the patterns in this guide are designed to make harder:

1. **Direct dependency compromise** — a package you chose was hijacked, typosquatted, or maliciously updated. The 2018 `event-stream` incident is the canonical example: a transitive package was handed off to a new "maintainer" who shipped a bitcoin-stealing update. Any project without a lockfile picked it up on its next install.
2. **Transitive dependency compromise** — same as #1, but the bad package is one you didn't choose; it came along with something else. The blast radius is wider and the detection lag is longer.
3. **Registry-level tampering** — the registry (npm, PyPI, crates.io, Docker Hub) serves a tampered tarball, either through namespace hijack, account takeover, or CDN compromise.
4. **CI/CD action compromise** — a GitHub Action your workflow references at `@v3` gets a new tag pointed at a malicious commit. The 2025 `tj-actions/changed-files` incident affected thousands of repositories simultaneously by abusing exactly this surface.
5. **Build-time tooling compromise** — your compiler, bundler, or codegen tool emits malicious output. Source code looks clean; the artifact doesn't.
6. **Build environment compromise** — the runner host (often shared CI infrastructure) is tampered with between checkout and artifact upload.

Tier 1 closes most of #2 and the easy parts of #1 and #4 by removing ambiguity from what gets installed and what runs your build. Tier 2 adds active CI enforcement against #1, #3, and the harder parts of #4. Tier 3 produces verifiable build attestations so #5 and #6 become detectable by consumers — not just by the project itself.

## How to choose your starting tier

| If your repo today... | Start at |
| --- | --- |
| Has no lockfile in any ecosystem, uses `@v3`-style action references, runs CI on `ubuntu-latest` | **Tier 1** |
| Is cleanly on Tier 1 but has no enforced check that blocks Tier 1 regressions in PRs | **Tier 2** |
| Has Tier 2 in place but ships releases without provenance attestations or signed artifacts | **Tier 3** |
| Already does SLSA Build L3, publishes `--provenance` to registries, and has OpenSSF Scorecard ≥ 7 | You're past the scope of this guide |

Rough effort estimate per tier on a typical mid-sized OSS project:

- **Tier 1** — about an hour. Almost all mechanical.
- **Tier 2** — a week of measured rollout (start `deterministic-deps` in advisory mode for at least one release cycle so the team sees what enforcement would block before it blocks).
- **Tier 3** — a focused two-week effort, or a longer background rewrite of the release flow. The new artifacts (provenance, SBOM, signatures) typically require coordinating with whoever consumes your releases.

## How each pattern is structured

Every pattern in Tiers 1–3 answers the same questions in the same order, so you can scan or deep-read:

1. **Rule** — one sentence.
2. **Why it matters** — the attack it prevents, with a CVE or incident link where one exists.
3. **How to do it** — copy-pasteable config, tabbed where ecosystems differ.
4. **How to verify** — the `deterministic-deps` rule ID, alternative tools, and a manual one-liner for the no-CI case.
5. **Common pitfalls** — the 1–3 gotchas that bite people.
6. **Real example** — a commit-SHA permalink to a file in a public Ozark-Security-Labs repo demonstrating it.

## Where to file issues and PRs

This guide lives at <https://github.com/Ozark-Security-Labs/supply-chain-hardening>. Each rendered page has an "Edit this page" link that opens the underlying markdown file on GitHub. PRs are welcome — see [CONTRIBUTING](https://github.com/Ozark-Security-Labs/supply-chain-hardening/blob/main/CONTRIBUTING.md) for the per-pattern entry template and the PR checklist.
