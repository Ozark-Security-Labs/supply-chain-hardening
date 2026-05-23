# Tier 3 — Production

Tier 3 produces verifiable releases — a third party can confirm what built each artifact, with which inputs, from which commit, and that the artifact hasn't been tampered with since. This is the level at which a downstream consumer can meaningfully audit you without re-running your build.

You should be cleanly on [Tier 2 — Hardened](../tier-2-hardened/) before adopting Tier 3. Tier 3 patterns assume the Tier 1–2 disciplines are in place; some (3.1, 3.4, 3.5) won't function correctly without them.

## Patterns

1. [Generate SLSA Build L3 provenance for every release](./slsa-provenance.md) — `slsa-framework/slsa-github-generator` reusable workflow, sigstore-signed attestation, verifier-friendly subjects
2. [Generate and attach an SBOM at release time](./sbom.md) — CycloneDX or SPDX, per-language tooling, checksum sidecar
3. [Sign release artifacts with sigstore / cosign (keyless)](./signed-artifacts.md) — OIDC-rooted signatures, no long-lived private keys, transparency-log-backed
4. [Use OIDC for cloud and registry pushes](./oidc-for-cloud-and-registries.md) — federated trust replacing static API keys for AWS / GCP / Azure / npm / PyPI Trusted Publishers
5. [Publish with registry-native provenance](./registry-provenance.md) — `npm publish --provenance`, PyPI PEP 740 attestations, crates.io trusted publishing
6. [Publish an OpenSSF Scorecard score](./scorecard-public-score.md) — automated posture rollup, badged in the README, public on scorecard.dev
7. [Protect the default branch and release tags with rulesets](./branch-and-tag-protection.md) — JSON-defined, source-controlled, including signature requirements and tag-update blocks

## Worked example

[`Ozark-Security-Labs/SessionScope`](https://github.com/Ozark-Security-Labs/SessionScope) is the most mature Tier 3 release pipeline in the OSL portfolio: tag-protection check, multi-platform build, **reproducible-build verification (cold + warm, byte-identical)**, CycloneDX SBOM, SLSA generic_slsa3 provenance via sigstore, checksum sidecars for every artifact, CHANGELOG-extracted release notes — all stitched together with explicit `needs:` dependencies between jobs.

[`Ozark-Security-Labs/PkgWarden`](https://github.com/Ozark-Security-Labs/PkgWarden) demonstrates the branch-protection-ruleset side (Rule 3.7) cleanly: `.github/rulesets/main-protection.json` committed in the repo, applied to the default branch with signed commits required and the `repository hygiene` status check gating merges.

Three patterns currently lack an in-tree OSL real example and link to canonical external references instead:

- **Rule 3.3 (cosign signed artifacts)** — no OSL project uses standalone `cosign sign-blob` today; SLSA provenance covers most of the same trust surface via its sigstore-signed attestation
- **Rule 3.4 (OIDC for cloud / registry)** — no OSL release pipeline federates to a cloud provider today
- **Rule 3.5 (registry-native provenance)** — no OSL npm publish flow uses `--provenance` today; the osl-* TypeScript forks predate npm Trusted Publishers

Closing those gaps is a near-term follow-up; the underlying infrastructure (sigstore via SLSA, `id-token: write` already in workflows) is mostly in place.

## After Tier 3

Tier 3 is the recommended terminal state for production OSS. Beyond it lies:

- **Reproducible builds** — covered briefly in Rule 3.1's "going further" section; SessionScope demonstrates the in-CI verification pattern (F-26)
- **Full SLSA Build L4** — adds hermetic builds, isolated builders. Currently impractical for most projects without dedicated infrastructure
- **Ecosystem-specific deeper hardening** — out of scope for this guide
