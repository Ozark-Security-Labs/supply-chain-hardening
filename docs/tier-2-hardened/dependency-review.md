# Rule 2.4 — Run Dependency Review on every PR

> Tier 2 · Hardened

## Rule

Every pull request runs the [Dependency Review action](https://github.com/actions/dependency-review-action) against the diff. The action fails the PR when the changeset introduces vulnerable, malicious, or license-incompatible dependencies.

## Why it matters

Tier 1's lockfile commitment makes installs deterministic. It doesn't tell you anything about whether the dependencies in that lockfile are *safe to install*. Dependency Review reads the manifest changes in a PR, asks GitHub's [Advisory Database](https://github.com/advisories) about the added or upgraded packages, and reports back: vulnerabilities at-or-above your configured severity threshold, license violations against your allow- or deny-lists, and (when available) reputation signals from the [OpenSSF Package Repository](https://api.deps.dev/) feed.

The check runs only on the diff, not on the whole tree. That keeps it fast (seconds) and means you don't have to fix every pre-existing vulnerability before turning it on — only stop adding new ones. For supply-chain integrity it closes the gap between "we have a lockfile" (Rule 1.1) and "we know the lockfile entries aren't already known-bad" (this rule).

Dependency Review requires the GitHub Dependency Graph to be enabled on the repository. For public repos this is the default; for private repos and some org-default-disabled cases, you'll need to flip the toggle at **Settings → Code security and analysis → Dependency graph** (and at the org level, if you control it).

## How to do it

A minimal workflow at `.github/workflows/dependency-review.yml`:

```yaml
name: dependency review

on:
  pull_request:

permissions:
  contents: read

jobs:
  review:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v5.0.0
        with:
          persist-credentials: false
      - uses: actions/dependency-review-action@2031cfc080254a8a887f58cffee85186f0e49e48  # v4.x
        with:
          fail-on-severity: moderate
          comment-summary-in-pr: on-failure
```

Two configuration choices matter:

**`fail-on-severity`** — `moderate` is a reasonable default. `high` lets through too much; `low` blocks transient false-positive churn from advisories that get re-classified after publication. Tighten over time as the team gets comfortable.

**`comment-summary-in-pr`** — `on-failure` keeps PR conversations clean while still surfacing the explanation when something blocks. `always` is noisier; `never` makes the failure cryptic.

### License gates (optional)

If your org has a license policy, add it explicitly:

```yaml
      - uses: actions/dependency-review-action@<sha>
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-3.0, AGPL-3.0
          # OR (alternatively, not in addition):
          allow-licenses: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause
```

Pick `deny-licenses` *or* `allow-licenses`, not both. Allow-list is stricter; deny-list is more permissive and easier to maintain when the project's own license is permissive.

### Gating on the right event

The action only runs on `pull_request` events — it needs the base/head diff. If your security workflow runs on both PR and push, gate the dependency-review job with `if: github.event_name == 'pull_request'`:

```yaml
jobs:
  dependency-review:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@<sha>
      - uses: actions/dependency-review-action@<sha>
```

## How to verify

- **`deterministic-deps`** does not duplicate this check — vulnerability and license data live in GitHub's Advisory Database, not in static analysis of files in the tree. The two checks are complementary.
- **Alternatives:** [`pip-audit`](https://github.com/pypa/pip-audit), [`npm audit`](https://docs.npmjs.com/cli/v10/commands/npm-audit), [`cargo audit`](https://github.com/rustsec/rustsec/tree/main/cargo-audit) report against the same ecosystem advisory data; OSV-Scanner and Trivy cover a broader artifact surface. Dependency Review is the GitHub-native option and the only one that integrates directly with the PR diff and the Advisory Database.
- **Manual:** the [`gh dependency-graph`](https://cli.github.com/manual/gh_api) and [GraphQL API](https://docs.github.com/en/graphql/reference/objects#dependencygraphmanifest) endpoints return the same data Dependency Review consumes; useful for one-off audits outside of CI.

## Common pitfalls

1. **Running on `push` or `schedule` and seeing the action fail with "Dependency Review is not supported on this event."** The action is PR-only. Either move it to its own workflow on `pull_request`, or gate the job with `if: github.event_name == 'pull_request'`.
2. **Dependency Graph disabled at the org level.** The action fails with a "not supported on this repository" error. Fix at **Org → Settings → Code security and analysis → Dependency graph**, then verify per-repo at **Repo → Settings → Code security and analysis**.
3. **Setting `fail-on-severity: low` on a freshly-wired repo.** Existing dependencies bring existing advisories; you'll get blocked on the first PR by something pre-existing. Start at `moderate`, tighten when the noise is zero.
4. **License gates without a project policy.** `deny-licenses: GPL-3.0` blocks a PR but doesn't explain why beyond the action output. Document the license policy in `CONTRIBUTING.md` so contributors see the rule before they hit the gate.
5. **Forgetting that fork PRs run with reduced token scope.** Dependency Review works for fork PRs but with read-only access. Confirm by opening a PR from a fork once.

## Real example

[`Ozark-Security-Labs/SessionScope/.github/workflows/security.yml`](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/security.yml) gates `dependency-review` to `pull_request` events only and pairs it with a `cargo-audit` job (a Rust-specific advisory check) on the schedule. Setting both up alongside each other is the typical hardened pattern.
