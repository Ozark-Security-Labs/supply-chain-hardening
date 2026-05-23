# Rule 3.3 — Sign release artifacts with sigstore / cosign (keyless)

> Tier 3 · Production

## Rule

Every release artifact carries a sigstore signature created during the release workflow using GitHub Actions OIDC (keyless mode). Consumers verify the signature against the workflow identity that produced it.

## Why it matters

SLSA provenance (Rule 3.1) attests *what built the artifact*. SBOMs (Rule 3.2) attest *what's inside the artifact*. Neither attests *that this binary is the one you signed*. For the third question — "was this file tampered with after it left the build runner?" — you need an explicit artifact signature.

Historically, signing meant managing a PGP private key, rotating it, and explaining to contributors how to keep it safe. Sigstore changed that: signing happens against a short-lived certificate tied to the workflow's GitHub Actions OIDC identity, the signature is logged to the public Rekor transparency log, and verification looks up the workflow identity in Rekor. There is no long-lived private key, no rotation, and no key-distribution problem — only the workflow's identity (`https://github.com/your-org/your-repo/.github/workflows/release.yml@refs/tags/v1.2.3`) and a Rekor log entry.

For supply-chain integrity, this fills the gap between SLSA provenance and artifact identity: a SLSA attestation says "this build produced these hashes"; a sigstore signature says "this exact byte sequence was signed by that build." Together they form a verifiable chain.

## How to do it

The minimal pattern using [`cosign`](https://github.com/sigstore/cosign):

```yaml
jobs:
  build:
    # ... produces dist/*.tar.gz etc. ...

  sign:
    needs: build
    runs-on: ubuntu-24.04
    permissions:
      id-token: write      # OIDC for keyless signing
      contents: write      # upload signatures to release
    steps:
      - uses: actions/checkout@<sha>
        with:
          ref: ${{ github.ref_name }}

      - name: Download release artifacts
        uses: actions/download-artifact@<sha>
        with:
          pattern: release-*
          path: dist
          merge-multiple: true

      - uses: sigstore/cosign-installer@<sha>
        with:
          cosign-release: 'v2.4.1'

      - name: Sign each artifact (keyless)
        working-directory: dist
        run: |
          set -euo pipefail
          for f in *.tar.gz *.zip; do
            [ -f "$f" ] || continue
            cosign sign-blob --yes \
              --output-certificate "$f.cert" \
              --output-signature   "$f.sig" \
              "$f"
          done

      - uses: softprops/action-gh-release@<sha>
        with:
          files: |
            dist/*.cert
            dist/*.sig
```

Three things to know:

**`cosign sign-blob` not `cosign sign`.** `cosign sign` is for container images stored in a registry that supports OCI signature uploads. `cosign sign-blob` is for arbitrary files (tarballs, zips, SBOMs, anything) — the signature and certificate are written as separate `.sig` and `.cert` files alongside the artifact.

**`--yes` skips the interactive prompt** asking whether you really want to sign. Required in CI; the prompt is for human protection against accidentally signing the wrong thing locally.

**Keyless mode is the default in cosign 2.x.** No `--key` flag means "use the OIDC identity from the environment." On GitHub Actions, that resolves to the workflow's identity automatically when `id-token: write` is in the job's permissions.

### Consumers verify with cosign

```sh
cosign verify-blob \
  --certificate-identity 'https://github.com/your-org/your-repo/.github/workflows/release.yml@refs/tags/v1.2.3' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  --signature 'my-project-1.2.3.tar.gz.sig' \
  --certificate 'my-project-1.2.3.tar.gz.cert' \
  my-project-1.2.3.tar.gz
```

The `--certificate-identity` is the canonical place to specify what you're verifying against. Document the exact string in your release notes — it's the bind between "this signature is valid" and "this signature is *the one I expected*."

### Combine with SLSA

SLSA's `slsa-github-generator` reusable workflow signs the provenance attestation via sigstore the same way. If you're already wiring SLSA, you have OIDC `id-token: write` configured and a sigstore signing flow at runtime — adding a separate cosign sign job is incremental. Some projects skip standalone cosign entirely if they ship SLSA + reproducible builds + SBOMs, since the provenance covers the "what was built" question and reproducibility lets consumers re-derive the hash independently. Pragmatic choice; the cosign signature gives you a *separate* verification path that doesn't require reproducing the build.

## How to verify

- **`deterministic-deps`** doesn't enforce signing — it's a release-time discipline, not a static configuration concern.
- **Alternatives:**
  - [GitHub Artifact Attestations](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations) (`actions/attest`) wraps sigstore signing as a GitHub-native action, producing a `.attestation.json` file with the same trust roots
  - PGP signing is still available but increasingly seen as deprecated for new projects — no transparency log, key-management overhead
- **Manual:** verify a released artifact yourself:

  ```sh
  gh release download v1.2.3 --pattern '*.tar.gz' --pattern '*.sig' --pattern '*.cert'
  cosign verify-blob \
    --certificate-identity 'https://github.com/your-org/your-repo/.github/workflows/release.yml@refs/tags/v1.2.3' \
    --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
    --signature *.sig \
    --certificate *.cert \
    *.tar.gz
  ```

## Common pitfalls

1. **Forgetting `id-token: write`.** Keyless signing depends on the GitHub OIDC token. The error ("no OIDC issuer configured") is opaque if you don't know what you're looking for.
2. **Using `cosign sign` instead of `cosign sign-blob` for tarballs.** `cosign sign` expects an OCI image reference. The error mentions OCI; the fix is the verb.
3. **Not documenting the `--certificate-identity` string.** Consumers can't verify without knowing the exact workflow identity. Put it in release notes, an `INSTALL.md`, or the README.
4. **Pinning an old cosign version that uses a different signing flow.** cosign 1.x and 2.x differ in defaults (1.x required `--keyless` explicitly; 2.x is keyless by default). Pin to 2.x; treat upgrades as a release-flow review event.
5. **Confusing cosign signatures with SLSA provenance.** They're different artifacts with different verification commands and different identity assertions. Document both if you ship both; document which is canonical for your project.

## Real example

No public Ozark-Security-Labs project currently uses standalone `cosign sign-blob` on release artifacts; the org's release pipelines rely on SLSA provenance (which is sigstore-signed under the hood by `slsa-github-generator`) plus reproducible builds for verifiable identity. For a strong external example, see [`sigstore/cosign`'s own release workflow](https://github.com/sigstore/cosign/tree/main/.github/workflows) — they sign their own releases this way as the canonical reference.

Adding a standalone `cosign sign-blob` step to one of the existing OSL release pipelines (likely `SessionScope`, as the most mature release flow) is a near-term follow-up; the value is the *separate* verification path it provides on top of SLSA provenance.
