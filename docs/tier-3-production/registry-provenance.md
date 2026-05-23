# Rule 3.5 — Publish with registry-native provenance (`npm --provenance`, PyPI, crates.io)

> Tier 3 · Production

## Rule

When a release pushes to a public package registry, use the registry's native provenance flag so the published version carries a verifiable attestation tying it back to the workflow run that built it.

## Why it matters

Rules 3.1 (SLSA provenance) and 3.3 (cosign signing) produce attestations attached to your GitHub Release. A downstream consumer who fetches your tarball from `releases/v1.2.3` can verify them. A downstream consumer who runs `npm install` or `pip install` doesn't — they're talking to the registry, not your release page.

Registry-native provenance closes that gap. The registry stores an attestation alongside the published version; client tooling (`npm install --foreground-scripts` and `npm audit signatures`, `pip install` with PEP 740 verification, `cargo install`'s emerging support) can verify it on install. The end-user gets a verification path without ever knowing your release is on GitHub.

The mechanism under the hood is the same OIDC exchange as Rule 3.4: the workflow proves its identity via GitHub's OIDC issuer, the registry signs an attestation binding the version to that identity. No long-lived registry token, no separate sigstore CLI invocation; the registry itself wires the provenance into its tooling.

## How to do it

### npm — `--provenance`

Requires:

- Public repo on GitHub (the workflow identity must be publicly attestable)
- `id-token: write` permission on the publish job
- [npm Trusted Publisher](https://docs.npmjs.com/trusted-publishers) set up on the package side (configured at `npmjs.com/package/<name>/access`)

The workflow:

```yaml
jobs:
  publish:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@<sha>
      - uses: actions/setup-node@<sha>
        with:
          node-version-file: '.nvmrc'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm publish --provenance --access public
```

No `NPM_TOKEN`. No `secrets.NPM_TOKEN` reference. The OIDC token in `id-token: write` is exchanged with npm's registry for a short-lived publish credential; the published version's metadata then includes an `attestations` block pointing at sigstore + Rekor.

Consumers verify with:

```sh
npm audit signatures
```

Run inside any node project. Reports each installed package's provenance status (and warns on packages that ship without provenance once you'd expect it).

### PyPI — PEP 740 + `pypa/gh-action-pypi-publish`

Requires:

- Public repo on GitHub
- `id-token: write` on the publish job
- [PyPI Trusted Publisher](https://docs.pypi.org/trusted-publishers/) configured on the package's PyPI page

The workflow:

```yaml
jobs:
  publish:
    runs-on: ubuntu-24.04
    permissions:
      id-token: write
    steps:
      - uses: actions/checkout@<sha>
      - uses: actions/setup-python@<sha>
        with:
          python-version-file: '.python-version'
      - name: Build distribution
        run: |
          python -m pip install --upgrade build
          python -m build
      - uses: pypa/gh-action-pypi-publish@<sha>
        with:
          # Trusted Publisher provides credentials via OIDC; no token input.
          attestations: true
```

Consumers using a current `pip` version (24.3+) can opt into attestation verification with `pip install --require-hashes ... --use-feature=truststore`. Verification-by-default is planned but not yet shipped.

### crates.io — trusted publishing (rollout in progress)

Crates.io is rolling out trusted publishing through an RFC-led stabilisation track in 2025 (see [rust-lang/rfcs#3691](https://github.com/rust-lang/rfcs/pull/3691) and the [crates.io tracking issue](https://github.com/rust-lang/crates.io/issues/10396)). Once stable, `cargo publish` from a workflow with `id-token: write` and a configured trusted publisher on the crates.io side will work the same way as npm and PyPI.

Until then, `cargo publish --token "$CARGO_REGISTRY_TOKEN"` with a token in repo secrets is the only path. This rule's intent for Rust projects is: switch to trusted publishing as soon as it's available; in the meantime, keep the token tightly scoped (registry-only, no other permissions) and on a rotation cadence.

### GitHub Packages, Maven Central, RubyGems, Container Registries

| Registry | OIDC support | Notes |
| --- | --- | --- |
| GitHub Packages (npm, Maven, Docker, etc.) | Native | Uses `GITHUB_TOKEN` directly with the appropriate `packages: write` permission |
| Maven Central via Sonatype | Federated via [`sigstore-maven-plugin`](https://github.com/sigstore/sigstore-java) (in development); falls back to GPG signing today | Maven Central's sigstore path is rolling out gradually |
| RubyGems | Limited; trusted-publisher work in 2025 | Token-based today |
| GHCR / Docker Hub (containers) | Use `cosign sign` (Rule 3.3) against the published image reference | OCI image signatures are stored alongside the image in the registry |

The pattern across all of these: replace long-lived tokens with OIDC + sigstore-rooted trust where the registry supports it; sign separately with cosign where it doesn't.

## How to verify

- **`deterministic-deps`** doesn't cover this — it's a release-time mechanism, not a static finding.
- **Alternatives:** for npm specifically, [Socket](https://socket.dev/), [Snyk](https://snyk.io/), and other dep-security platforms surface provenance status alongside their other signals. For OSS at large, [`sigstore-go`](https://github.com/sigstore/sigstore-go) is the lib most third-party verifiers use.
- **Manual:** for npm,

  ```sh
  npm view your-package@1.2.3 --json | jq '.attestations'
  ```

  returns the attestation set if the version was published with `--provenance`. Empty or missing means it wasn't.

## Common pitfalls

1. **Forgetting Trusted Publisher configuration on the registry side.** The workflow looks right; the publish fails with "no OIDC trust configured." Configure once per package at the registry, then forever after the workflow flows through.
2. **`--provenance` on a private GitHub repo.** npm rejects this — the workflow identity must be publicly verifiable. Either make the repo public, or accept that the published version ships without provenance.
3. **Mixing `--provenance` and a static `NPM_TOKEN` in the same workflow.** The `--provenance` path is OIDC-only; a `NODE_AUTH_TOKEN` env var in scope confuses the publish step. Remove all token-based auth from the job.
4. **Pinning `pypa/gh-action-pypi-publish` to an older version that doesn't support `attestations: true`.** The action's PEP 740 support landed in 2024; pin to a recent SHA / release.
5. **Assuming `--provenance` on npm covers the whole tree.** It attests the specific version you published. Transitive dependencies are independent; their provenance status is their maintainers' choice.

## Real example

No public Ozark-Security-Labs project currently ships to a public registry with `--provenance`. The osl-* TypeScript forks (`osl-glob`, `osl-minimatch`, `osl-js-yaml`) publish to npm via a manual `npm publish` flow that predates Trusted Publishers; the action repo (`deterministic-deps`) publishes to the GitHub Marketplace where provenance is implicit in the marketplace listing. Migrating one of the npm publish flows to Trusted Publishers + `--provenance` is the canonical next step; the workflow change is small once the npmjs.com side is configured.

For a strong external example, see [`sigstore/cosign`'s npm publish workflow](https://github.com/sigstore/cosign/blob/main/.github/workflows/release.yml) (when it ships a JS distribution) or [`pypa/sampleproject`'s PyPI publish](https://github.com/pypa/sampleproject) for the PEP 740 pattern.
