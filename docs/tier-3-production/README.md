# Tier 3 — Production

> Status: outline only. Patterns ship in **v0.3.0** (see [project status](../../README.md#status)).

Tier 3 produces verifiable releases — a third party can confirm what built each artifact, with which inputs, from which commit. This is the level at which a downstream consumer can meaningfully audit you.

Worked example throughout this tier: [`Ozark-Security-Labs/SessionScope`](https://github.com/Ozark-Security-Labs/SessionScope).

## Planned patterns

1. **SLSA Build L3** — `slsa-framework/slsa-github-generator` for provenance attestations
2. **SBOM at build time** — `cyclonedx` or `syft`, attached to the release with a checksum
3. **Signed artifacts** — sigstore / cosign keyless signing tied to the OIDC identity
4. **OIDC for cloud and registry pushes** — no long-lived secrets in repo, anywhere
5. **`npm publish --provenance` / cargo provenance** — registry-level attestation chain
6. **OpenSSF Scorecard public score** — automated reproducible posture check, badged in the README
7. **Release tag + branch protection + required reviewers** — codified in repo rulesets, not just maintainer habit

## Prerequisites

You should be cleanly on [Tier 2 — Hardened](../tier-2-hardened/) before adopting Tier 3.

## After Tier 3

Tier 3 is the recommended terminal state for production OSS. Beyond it lies reproducible builds (covered briefly in the [verification cookbook](../appendix/verification-cookbook.md)) and ecosystem-specific deeper hardening that this guide deliberately does not catalogue.
