# Rule 3.1 — Generate SLSA Build L3 provenance for every release

> Tier 3 · Production

## Rule

Every release artifact is accompanied by a [SLSA Build Level 3](https://slsa.dev/spec/v1.0/levels#build-l3) provenance attestation that names the source commit, the builder workflow, and the inputs that produced the artifact. The standard path on GitHub is to invoke `slsa-framework/slsa-github-generator`'s reusable workflow from your release workflow.

## Why it matters

A downstream consumer pulling your release tarball today has no cryptographic way to answer "what built this?" The release page might say `v1.2.3 of org/repo` but that's just a label — there's no link back to the source commit, the workflow run, or the toolchain version that produced the binary. SLSA provenance fills that gap with a signed, in-toto attestation: a JSON document, signed by sigstore via the GitHub Actions OIDC identity, that names every relevant input to the build.

Tier 2 made your build inputs auditable to *you*. Tier 3 makes them auditable to anyone downstream. The SLSA verifier (`slsa-verifier`) takes a release artifact and its `.intoto.jsonl` file and confirms:

- The artifact's SHA-256 matches the subject in the attestation
- The attestation was signed by the GitHub Actions workflow identity claimed
- The workflow ran from a pinned tag of `slsa-framework/slsa-github-generator`
- The source repo matches the expected `org/repo`

If any of those checks fail, the artifact is rejected. This is what "verifiable releases" means in practice.

## How to do it

SLSA Build L3 on GitHub is achieved by invoking the SLSA generator reusable workflow from your release workflow. A minimal pattern, factored from the SessionScope release.yml:

```yaml
name: release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-24.04
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ github.ref_name }}
      # ... build steps that produce dist/<artifact> ...
      - name: Generate SHA-256 sidecars and SLSA subjects
        id: hash
        shell: bash
        working-directory: dist
        run: |
          set -euo pipefail
          for artifact in *.tar.gz *.zip; do
            [ -f "$artifact" ] && sha256sum "$artifact" > "$artifact.sha256"
          done
          hashes="$(sha256sum *.tar.gz *.zip 2>/dev/null | base64 -w0)"
          echo "hashes=${hashes}" >> "${GITHUB_OUTPUT}"

  provenance:
    name: Generate SLSA provenance
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: write
    # The generic SLSA reusable workflow requires a semantic version ref.
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.1.0
    with:
      base64-subjects: ${{ needs.build.outputs.hashes }}
      upload-assets: true
      provenance-name: my-project-${{ github.ref_name }}.intoto.jsonl
```

Three things worth calling out:

**The reusable workflow MUST be referenced by tag, not SHA.** This contradicts Rule 1.2 ("SHA-pin every third-party action") and is the canonical example of when an allowlist entry in `.deterministic-deps.yml` is required. The SLSA verifier confirms the workflow ran from a signed tag of the generator; if you SHA-pin instead, verification fails. Add:

```yaml
allowlist:
  - file: .github/workflows/release.yml
    ruleId: github-actions/sha-pin
```

**The `hashes` output is base64-encoded.** The SLSA generator takes a base64 of the `sha256sum`-formatted lines and uses it as the subject set. Generate it once in the build job; the provenance job consumes it via `needs.build.outputs.hashes`.

**`upload-assets: true`** attaches the `.intoto.jsonl` file directly to the GitHub Release. Downstream consumers `gh release download` it next to the tarball.

### Verifying the attestation

Consumers verify with [`slsa-verifier`](https://github.com/slsa-framework/slsa-verifier):

```sh
slsa-verifier verify-artifact \
  --provenance-path my-project-v1.2.3.intoto.jsonl \
  --source-uri github.com/your-org/your-repo \
  --source-tag v1.2.3 \
  my-project-1.2.3-linux-x86_64.tar.gz
```

Document this command in your release notes or `INSTALL.md` so consumers know how to use what you produced.

### Going further: reproducible builds

SLSA L3 doesn't require reproducibility, but a reproducible build adds an independent verification path: any third party can rebuild from the same commit + same toolchain and compare hashes. SessionScope's release pipeline ([F-26](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/release.yml)) verifies this in CI by doing two builds on the same runner (cold + warm `target/`) and asserting byte-identical SHA-256 output. The pattern is straightforward once Rule 1.4 (toolchain pin) and `SOURCE_DATE_EPOCH` discipline are in place; full reproducibility across runners is harder and ecosystem-specific (deterministic linker flags, embedded timestamps, etc.).

## How to verify

- **`deterministic-deps`** doesn't produce SLSA attestations — it's the static-check side. Use it to confirm the release workflow's *own* inputs are pinned correctly.
- **Alternatives:** [GitHub Artifact Attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations) (`actions/attest-build-provenance`) is GitHub's lighter-weight first-party version of SLSA-style attestations. It's less standardised than SLSA but easier to wire up; consider it for projects that don't need cross-platform verifier compatibility. [`in-toto`](https://in-toto.io/) is the underlying attestation framework; SLSA is one of several profiles built on it.
- **Manual:** after a release ships, fetch `*.intoto.jsonl` and inspect with `cat | jq .` to confirm the predicate's `buildDefinition.resolvedDependencies` includes your source commit and the SLSA generator version.

## Common pitfalls

1. **SHA-pinning the SLSA generator.** Breaks verification. Tag reference is required; allowlist the deterministic-deps rule.
2. **Forgetting `id-token: write` on the provenance job.** SLSA signs via GitHub Actions OIDC; without the scope, the OIDC token request fails with a confusing 403.
3. **Pinning a SLSA generator pre-v2.x.** v1.x generators are EOL'd and produce attestations that current `slsa-verifier` may not accept. Use `@v2.1.0` or whatever the current stable release is.
4. **Multiple artifacts but one hash subject.** The `base64-subjects` input takes the *concatenated* `sha256sum` output across every artifact you want covered. Producing a partial list silently signs only those artifacts; consumers verifying others fail with "subject not found."
5. **Treating SLSA as enough for binary verification.** SLSA attests the *build*; cosign signing (Rule 3.3) attests the *artifact* identity. They're complementary; large projects do both.

## Real example

[`Ozark-Security-Labs/SessionScope/.github/workflows/release.yml`](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/release.yml) is a worked Tier 3 release pipeline: tag-protection check, multi-platform build, byte-identical reproducible-build verification, CycloneDX SBOM (Rule 3.2), SLSA generic_slsa3 provenance, checksum sidecars, and CHANGELOG-extracted release notes — all wired together with explicit `needs:` between jobs. Read it as one cohesive Tier 3 pattern, not a sum of seven rules in isolation.
