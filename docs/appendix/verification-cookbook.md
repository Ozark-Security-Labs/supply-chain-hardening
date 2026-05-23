# Appendix E — Verification cookbook

For each pattern in Tiers 1–3, the one-liner that confirms it's in place — `gh api`, `grep`, `jq`, or a single CLI invocation. The reviewer's rubric (Appendix D) summarises *what* to check; this appendix is the exact command, ready to paste.

Most commands assume you're in a working directory checked out from the repo you're verifying. Where a command queries GitHub APIs, replace `<org>/<repo>` with the target.

## Tier 1 — Baseline

```sh
# 1.1 Lockfile present (npm)
[ -f package-lock.json ] && jq -e '.lockfileVersion >= 3' package-lock.json >/dev/null

# 1.1 Lockfile present (Python with hashes)
! grep -vE '^\s*(#|--|$)' requirements.txt 2>/dev/null | grep -v -- '--hash=sha256:'

# 1.1 Lockfile present (Go)
[ -f go.mod ] && [ -f go.sum ]

# 1.1 Lockfile present (Rust)
[ -f Cargo.toml ] && [ -f Cargo.lock ]

# 1.2 All third-party actions SHA-pinned
! grep -rEn '^\s*-?\s*uses:\s*[^./@][^@]*@[^[:space:]]+' .github/workflows/ \
  | grep -vE '@[0-9a-f]{40}(\s|$|#)'

# 1.3 No floating runner labels
! grep -rEn '^\s*runs-on:\s*([a-z]+-latest|\[[^]]*-latest)' .github/workflows/

# 1.4 Language toolchain pinned (any of these)
[ -f .nvmrc ] && grep -qE '^v?[0-9]+\.[0-9]+\.[0-9]+$' .nvmrc
[ -f .python-version ] && grep -qE '^[0-9]+\.[0-9]+\.[0-9]+$' .python-version
[ -f rust-toolchain.toml ] && grep -qE '^\s*channel\s*=\s*"[0-9]+\.[0-9]+\.[0-9]+"' rust-toolchain.toml

# 1.5 Every workflow has a top-level permissions block
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
  [ -f "$f" ] || continue
  head -30 "$f" | grep -qE '^permissions:' || echo "$f: no top-level permissions"
done
```

## Tier 2 — Hardened

```sh
# 2.1 deterministic-deps in enforce mode at low threshold
grep -A 4 'deterministic-deps@' .github/workflows/*.yml | grep -E "mode:\s*enforce" \
  && grep -A 4 'deterministic-deps@' .github/workflows/*.yml | grep -E "severity-threshold:\s*low"

# 2.1 Run locally to dry-test (any project)
npx -y Ozark-Security-Labs/deterministic-deps@v1 --mode advisory

# 2.2 Python install uses --require-hashes in CI
grep -rEn 'pip install.*--require-hashes' .github/workflows/

# 2.3 Every Dockerfile FROM has a sha256 digest
! grep -rEn '^FROM\s+[^[:space:]]+(\s|$)' Dockerfile docker/*.Dockerfile 2>/dev/null \
  | grep -v '@sha256:'

# 2.4 Dependency Review wired
grep -rl 'dependency-review-action' .github/workflows/

# 2.5 Active bot config
[ -f .github/dependabot.yml ] || [ -f renovate.json ] || [ -f .renovaterc.json ]

# 2.5 Recent bot PRs (last 30 days)
gh pr list --repo <org>/<repo> --state all --limit 50 --json author,createdAt \
  | jq '[.[] | select(.author.login == "dependabot[bot]" or .author.login == "renovate[bot]") | select(.createdAt > (now - 60*60*24*30 | todate))] | length'

# 2.6 persist-credentials: false on every checkout
! grep -rB 1 'actions/checkout@' .github/workflows/ | grep -A 3 'uses:' \
  | grep -v 'persist-credentials: false' | grep -q 'uses: actions/checkout'

# 2.6 harden-runner wired
grep -rl 'step-security/harden-runner' .github/workflows/

# 2.7 CodeQL wired
grep -rl 'codeql-action/init\|codeql-action/analyze' .github/workflows/
```

## Tier 3 — Production

```sh
# 3.1 SLSA provenance attached to a specific release
gh release view <tag> --repo <org>/<repo> --json assets -q '.assets[].name' | grep '.intoto.jsonl'

# 3.1 SLSA provenance contains the expected source repo and commit
gh release download <tag> --repo <org>/<repo> --pattern '*.intoto.jsonl'
jq '.payload | @base64d | fromjson | .subject, .predicate.buildDefinition.externalParameters' *.intoto.jsonl

# 3.1 Verify with slsa-verifier
slsa-verifier verify-artifact \
  --provenance-path <name>.intoto.jsonl \
  --source-uri github.com/<org>/<repo> \
  --source-tag <tag> \
  <artifact.tar.gz>

# 3.2 SBOM attached
gh release view <tag> --repo <org>/<repo> --json assets -q '.assets[].name' | grep -E '\.(cdx|spdx)\.json$'

# 3.2 SBOM checksum verifies
gh release download <tag> --repo <org>/<repo> --pattern '*.cdx.json' --pattern '*.cdx.json.sha256'
sha256sum -c *.cdx.json.sha256

# 3.2 SBOM component count (sanity check)
jq '.components | length' *.cdx.json

# 3.3 Cosign signatures attached
gh release view <tag> --repo <org>/<repo> --json assets -q '.assets[].name' | grep -E '\.(sig|cert)$'

# 3.3 Verify with cosign
cosign verify-blob \
  --certificate-identity 'https://github.com/<org>/<repo>/.github/workflows/release.yml@refs/tags/<tag>' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  --signature <name>.sig \
  --certificate <name>.cert \
  <artifact.tar.gz>

# 3.4 OIDC permissions present
grep -rn 'id-token: write' .github/workflows/

# 3.4 No long-lived registry tokens in secrets
gh api /repos/<org>/<repo>/actions/secrets --jq '.secrets[] | .name' | grep -iE '(npm|pypi|cargo|crates|docker|aws|gcp|azure)_(token|key|secret)'

# 3.5 npm package's most recent version has provenance
npm view <package>@<version> --json | jq '.attestations'

# 3.6 Scorecard public score
curl -s "https://api.scorecard.dev/projects/github.com/<org>/<repo>" | jq '.score'

# 3.7 Branch and tag rulesets active
gh api /repos/<org>/<repo>/rulesets --jq '.[] | {name, target, enforcement}'

# 3.7 Specific main-protection ruleset details
gh api /repos/<org>/<repo>/rulesets/<id> | jq '.rules'
```

## Artifacts you should expect to see at a Tier 3 release

A complete Tier 3 release attaches roughly this set of files to its GitHub Release. Treat missing files as gaps in the project's Tier 3 adoption.

| Artifact | What it is |
| --- | --- |
| `<name>-<version>.tar.gz` (or `.zip` / per-platform binary) | The release artifact |
| `<name>-<version>.tar.gz.sha256` | Checksum sidecar, verifies the artifact byte-for-byte |
| `<name>-<version>.intoto.jsonl` | SLSA provenance attestation (Rule 3.1) |
| `<name>-<version>.cdx.json` | CycloneDX SBOM (Rule 3.2) |
| `<name>-<version>.cdx.json.sha256` | SBOM checksum sidecar |
| `<name>-<version>.tar.gz.sig` and `.cert` | Cosign signature + certificate (Rule 3.3) |
| `SHA256SUMS` | Aggregate checksums for the whole asset set |
| Release notes referencing the changelog | Human-readable change summary |

A typical SessionScope release (the cross-tier worked example) attaches the per-platform tarballs, `.sha256` sidecars, the `.cdx.json` SBOM with its sidecar, and the `.intoto.jsonl` provenance. The cosign-specific `.sig` / `.cert` files are not attached separately — the SLSA generator's sigstore signature on the provenance covers the same trust surface for projects in that release-flow shape.

## When the cookbook isn't enough

These commands answer "is the pattern wired up?" — they don't answer "is the pattern wired up *correctly* for this project's threat model?" For deeper review:

- Read the project's `CONTRIBUTING.md` for the gating discipline (do reviewers actually require the checks to pass before merging?)
- Skim the last 20 closed PRs to see whether CI was historically green pre-merge
- Look at the commit history of `.github/workflows/` — frequent churn suggests the patterns were retrofitted recently and may not be fully internalised yet
- Open the most recent Scorecard report and read the per-check breakdown rather than just the aggregate

Posture commands are necessary; they're not sufficient.
