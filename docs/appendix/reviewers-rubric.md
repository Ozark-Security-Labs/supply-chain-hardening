# Appendix D — Reviewer's rubric

A one-page checklist for evaluating a third-party project's supply-chain posture. For each pattern in Tiers 1–3, names the file or artifact to look for, the command that confirms it, and what "good" looks like.

Use case: a security reviewer or procurement evaluator deciding whether an OSS dependency meets the posture bar for their environment, without re-reading every pattern in this guide.

## Tier 1 — Baseline

Every box checked means the project is at full Tier 1. Partial check is the typical starting state of an actively-maintained but not-yet-hardened repo.

| Rule | What to look for | Where | Command |
| --- | --- | --- | --- |
| 1.1 Lockfile present | `package-lock.json` (`lockfileVersion: 3`), `pnpm-lock.yaml`, `poetry.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, `gradle.lockfile`, etc. | Repo root or per-package | `ls package-lock.json Cargo.lock go.sum 2>/dev/null` |
| 1.2 Actions SHA-pinned | `uses: org/action@<40-char-sha>` for every third-party action | `.github/workflows/*.yml` | `grep -rE 'uses:\s+[^./@]+/[^@]+@[^[:space:]]+' .github/workflows/ \| grep -vE '@[0-9a-f]{40}'` should return nothing |
| 1.3 Runner OS pinned | `ubuntu-24.04`, `windows-2022`, `macos-14` (not `*-latest`) | `.github/workflows/*.yml` | `grep -rE 'runs-on:\s*([a-z]+-latest\|\[[^]]*-latest)' .github/workflows/` should return nothing |
| 1.4 Toolchain pinned | `.nvmrc`, `.python-version`, `rust-toolchain.toml`, `go.mod` toolchain triple | Repo root | `ls .nvmrc .python-version rust-toolchain.toml 2>/dev/null` |
| 1.5 Minimal `permissions:` | Top-level `permissions: { contents: read }` (or equivalent) on every workflow | `.github/workflows/*.yml` | `for f in .github/workflows/*.yml; do head -30 "$f" \| grep -qE '^permissions:' \|\| echo "$f: missing"; done` |

## Tier 2 — Hardened

| Rule | What to look for | Where | Command |
| --- | --- | --- | --- |
| 2.1 `deterministic-deps` enforced | `Ozark-Security-Labs/deterministic-deps` workflow in enforce mode at `severity-threshold: low` | `.github/workflows/dependency-determinism.yml` | `grep -A 3 'deterministic-deps@' .github/workflows/*.yml` |
| 2.2 Python `--hash=` enforced | Every `requirements*.txt` line has `--hash=sha256:`; install uses `--require-hashes` | Repo root | `! grep -vE '^\s*(#\|--\|$)' requirements.txt \| grep -v -- '--hash=sha256:'` |
| 2.3 Container digests | `FROM image:tag@sha256:<digest>` for every container ref | `Dockerfile`, `compose.yml`, `devcontainer.json`, `docker://` action uses | `grep -nE '^FROM\s+[^[:space:]]+(\s\|$)' Dockerfile \| grep -v '@sha256:'` should return nothing |
| 2.4 Dependency Review | `actions/dependency-review-action` step on every PR | `.github/workflows/*.yml` | `grep -rl 'dependency-review-action' .github/workflows/` |
| 2.5 Dependabot or Renovate | Active config; weekly PR cadence; `groups:` block or `packageRules` | `.github/dependabot.yml` or `renovate.json` | `[ -f .github/dependabot.yml ] \|\| [ -f renovate.json ]` + check the project's PR list for recent bot activity |
| 2.6 `persist-credentials: false` | Set on every `actions/checkout` except where a push is needed | `.github/workflows/*.yml` | `grep -A 3 'actions/checkout@' .github/workflows/*.yml \| grep -v 'persist-credentials: false' \| grep 'uses: actions/checkout' \|\| echo "all checkouts hardened"` |
| 2.7 CodeQL | `github/codeql-action/init` + `github/codeql-action/analyze` on PR and push | `.github/workflows/codeql.yml` | `grep -l 'codeql-action' .github/workflows/*.yml` |

## Tier 3 — Production

| Rule | What to look for | Where | Command |
| --- | --- | --- | --- |
| 3.1 SLSA provenance | `*.intoto.jsonl` file attached to GitHub Release; SLSA generator reusable workflow in release.yml | Release page + `.github/workflows/release.yml` | `gh release view <tag> --json assets -q '.assets[].name' \| grep intoto.jsonl` |
| 3.2 SBOM attached | `*.cdx.json` or `*.spdx.json` attached to GitHub Release with `.sha256` sidecar | Release page | `gh release view <tag> --json assets -q '.assets[].name' \| grep -E '(cdx\|spdx)\.json'` |
| 3.3 Cosign signature | `*.sig` and `*.cert` files attached to Release, OR SLSA provenance present (covers most of the same surface) | Release page | `gh release view <tag> --json assets -q '.assets[].name' \| grep -E '\.(sig\|cert)$'` |
| 3.4 OIDC for cloud / registry | `id-token: write` permission + OIDC-aware credentials action; no `*_TOKEN` / `*_KEY` in repo secrets for release | Workflow + repo settings | `grep -rn 'id-token: write' .github/workflows/`; check repo `Settings → Secrets and variables → Actions` |
| 3.5 Registry-native provenance | `--provenance` flag on `npm publish`, `attestations: true` on `pypa/gh-action-pypi-publish` | Publish workflow | `grep -rn 'provenance\|attestations:' .github/workflows/` |
| 3.6 Scorecard public score | OpenSSF Scorecard badge in README; score ≥ 7 on scorecard.dev | README + scorecard.dev | `curl -s https://api.scorecard.dev/projects/github.com/<org>/<repo>/badge \| grep -o 'score:[0-9.]*'` |
| 3.7 Branch + tag protection | `.github/rulesets/*.json` committed; ruleset active on default branch and `v*` tags | `.github/rulesets/` + GH API | `gh api /repos/<org>/<repo>/rulesets --jq '.[] \| {name, target, enforcement}'` |

## Scoring this rubric

This guide does not assign a numeric score across the rubric — different projects warrant different priorities. As a rough heuristic:

- **All Tier 1 boxes checked**: meets the floor for a dependency you'd accept into a production environment without further questions
- **All Tier 1 + Tier 2 boxes**: meets the bar for a project you'd actively recommend to peers
- **Tier 1 + 2 + significant Tier 3**: meets the bar for a project you'd cite as a reference example in a security-architecture review

Be wary of projects that have Tier 3 release attestations but missing Tier 1 baseline — it usually means the release theatre is more polished than the underlying CI, which is the wrong way around.

## When the rubric isn't enough

This rubric covers structural posture. It doesn't tell you whether the project's code is *correct*, whether its maintainers respond to security reports, whether its threat model matches yours, or whether the project will still be maintained in 18 months. For those questions, talk to people, read the issue tracker, and look at commit cadence. Posture is necessary, not sufficient.
