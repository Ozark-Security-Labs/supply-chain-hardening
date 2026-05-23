# Rule 3.2 — Generate and attach an SBOM at release time

> Tier 3 · Production

## Rule

Every release ships a Software Bill of Materials (SBOM) attached to the GitHub Release artifact set, in either [CycloneDX](https://cyclonedx.org/) or [SPDX](https://spdx.dev/) format. The SBOM is generated as part of the release workflow, has its own SHA-256 sidecar, and is referenced in the release notes.

## Why it matters

When a CVE drops against a transitive dependency, the question your downstream consumers will ask is: "are we affected?" Without an SBOM, the answer requires them to reproduce your build, parse your lockfile, and walk the dependency tree themselves — for every release they have deployed. With an SBOM attached to the release, the answer is `grep cve-affected-package sbom.json`.

For supply-chain hygiene, the SBOM is also the *static* record of what the build inputs were. Lockfiles record what *should* be installed; the SBOM records what *was* installed. If you don't ship one, your release notes' "no known vulnerabilities at build time" claim is unverifiable. If you do, a downstream auditor can re-check against the Advisory Database the day they discover the deployment.

## How to do it

Use the standard SBOM generator for your build's primary language and write the output to a workspace path the release job uploads:

| Ecosystem | Tool | Command |
| --- | --- | --- |
| Rust | [`cargo-cyclonedx`](https://github.com/CycloneDX/cyclonedx-rust-cargo) | `cargo cyclonedx --format json --all` |
| Node.js | [`@cyclonedx/cdxgen`](https://github.com/CycloneDX/cdxgen) | `cdxgen -o sbom.cdx.json` |
| Python | [`cyclonedx-bom`](https://pypi.org/project/cyclonedx-bom/) | `cyclonedx-py environment -o sbom.cdx.json` |
| Go | [`cyclonedx-gomod`](https://github.com/CycloneDX/cyclonedx-gomod) | `cyclonedx-gomod mod -licenses -json -output sbom.cdx.json` |
| Anything containerised | [`syft`](https://github.com/anchore/syft) | `syft <image-or-dir> -o cyclonedx-json=sbom.cdx.json` |
| Polyglot / source-of-truth | [`syft`](https://github.com/anchore/syft) | `syft dir:. -o spdx-json=sbom.spdx.json` |

For most projects, `syft` is the most ergonomic single-tool answer — it reads from many ecosystem manifests and produces either CycloneDX or SPDX format. For language-specific projects, the per-ecosystem tool above gives a richer SBOM (includes license fields, package URLs, declared vs resolved versions).

### Workflow pattern

A standalone SBOM job in your release workflow, factored from SessionScope's:

```yaml
sbom:
  name: Generate CycloneDX SBOM
  needs: prepare  # depends on the release-tag-validated context
  runs-on: ubuntu-24.04
  steps:
    - uses: actions/checkout@<sha>
      with:
        ref: ${{ needs.prepare.outputs.tag }}
    - uses: dtolnay/rust-toolchain@<sha>
      with:
        toolchain: "1.95"
    - name: Install cargo-cyclonedx
      run: cargo install cargo-cyclonedx --version 0.5.7 --locked
    - name: Generate CycloneDX SBOM
      env:
        VERSION: ${{ needs.prepare.outputs.version }}
      run: |
        set -euo pipefail
        mkdir -p sbom
        cargo cyclonedx --format json --all
        src="$(find crates/sessionscope-cli -maxdepth 2 -type f -name '*.cdx.json' | head -n 1)"
        cp "${src}" "sbom/sessionscope-${VERSION}.cdx.json"
    - name: Upload SBOM to workflow
      uses: actions/upload-artifact@<sha>
      with:
        name: release-sbom
        path: sbom/sessionscope-*.cdx.json
        retention-days: 7
        if-no-files-found: error
```

Then in the publish job, download the SBOM alongside the artifacts, generate a SHA-256 sidecar, and include it in `files:`:

```yaml
- name: Generate SHA-256 sidecar for SBOM
  working-directory: dist
  env:
    VERSION: ${{ needs.prepare.outputs.version }}
  run: |
    sbom="sessionscope-${VERSION}.cdx.json"
    sha256sum "${sbom}" > "${sbom}.sha256"
    sha256sum -c "${sbom}.sha256"

- uses: softprops/action-gh-release@<sha>
  with:
    files: |
      dist/*.tar.gz
      dist/*.zip
      dist/*.sha256
      dist/*.cdx.json
```

### Choosing a format

- **CycloneDX** (`*.cdx.json`) — favoured by OWASP, lighter schema, broader tooling for vulnerability cross-referencing
- **SPDX** (`*.spdx.json`) — favoured by the Linux Foundation, broader legal/compliance tooling

Pick one and stick with it. CycloneDX is the default in most language-specific generators; SPDX is more common when license compliance is the primary use case. The two are roughly equivalent; consumers can convert between them with `cdx2spdx` or `spdx-tools`.

### Pin the SBOM tool itself

`cargo install cargo-cyclonedx --version 0.5.7 --locked` — note `--version` (exact pin) and `--locked` (Cargo's frozen-lock equivalent). The SBOM tool is part of your trusted build base; don't `cargo install` it unpinned. The same discipline applies to `syft`, `cdxgen`, and friends — install at a pinned tag in a setup step.

## How to verify

- **`deterministic-deps`** doesn't enforce SBOM generation directly; the discipline is that the SBOM exists and is attached to the release. Verify by inspection of release assets.
- **Alternatives:**
  - [GitHub's dependency-graph SBOM export](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/exporting-a-software-bill-of-materials-for-your-repository) (`gh api .../dependency-graph/sbom`) — produces an SPDX SBOM from the dependency graph without running the build. Useful for repo-level reporting; less useful for release artifacts since it's not tied to a specific built version.
  - [`grype`](https://github.com/anchore/grype) — consumes the SBOM and reports known vulnerabilities. Pair with `syft` for the generate-then-scan pattern.
- **Manual:** download a release SBOM and confirm structure:

  ```sh
  gh release download v1.2.3 --pattern '*.cdx.json' --pattern '*.cdx.json.sha256'
  sha256sum -c *.cdx.json.sha256
  jq '.components | length' *.cdx.json   # sanity-check component count
  ```

## Common pitfalls

1. **Generating the SBOM from a different state than the build used.** `cargo cyclonedx` reads `Cargo.lock` at the time it runs; if you regenerate or update the lockfile between build and SBOM, the SBOM doesn't describe the released binary. Run SBOM generation in a job that checks out the *same* tag as the build, with no `cargo update`.
2. **Not pinning the SBOM tool itself.** `cargo install cargo-cyclonedx` (no `--version`) installs whatever is current; you get a different SBOM tool version per release. Pin it.
3. **Shipping the SBOM without a checksum sidecar.** Without `*.cdx.json.sha256`, consumers can't verify the SBOM wasn't tampered with in flight (or replaced on the release page). Treat SBOMs as first-class artifacts and sidecar them.
4. **Picking neither format consistently.** Producing CycloneDX one release and SPDX the next breaks downstream tooling that watches your release feed. Pick one at v1 and document the choice.
5. **Embedding secrets in the SBOM.** Some tools include URLs from git config or environment variables. Review the SBOM once before automating; redact private-registry URLs that include tokens (Rule from `deterministic-deps`' own redaction pass — same instinct applies here).

## Real example

[`Ozark-Security-Labs/SessionScope/.github/workflows/release.yml`](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/release.yml) (the `sbom` job, starting around line 234) generates a CycloneDX JSON via `cargo-cyclonedx --version 0.5.7 --locked`, uploads it as a workflow artifact, and the downstream publish job attaches it to the GitHub Release alongside a SHA-256 sidecar. The full release pipeline pairs the SBOM with SLSA provenance (Rule 3.1) and per-artifact checksums.
