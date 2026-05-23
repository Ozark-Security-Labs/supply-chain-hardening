# Rule 1.2 — SHA-pin every third-party GitHub Action

> Tier 1 · Baseline

## Rule

Reference every third-party GitHub Action by its full 40-character commit SHA, never by a tag, branch, or short SHA. Pin Docker image action references the same way, using `@sha256:<digest>`.

## Why it matters

Git tags are mutable. Anyone with push access to an action's repository can move `v3` to point at a different commit at any time. When your workflow says `uses: org/action@v3`, GitHub resolves that to whatever commit `v3` points at *the moment your build starts*, not the commit it pointed at when you wrote the workflow.

The canonical incident is the [March 2025 `tj-actions/changed-files` compromise](https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised). An attacker obtained a maintainer's token, retroactively repointed every version tag (`v1` through `v45.0.7`) to a single malicious commit, and that commit exfiltrated CI secrets from every running workflow that referenced the action by tag — thousands of repositories simultaneously. Repos that had SHA-pinned the same action were untouched; the malicious commit was not the one they had pinned, and `uses: org/action@<sha>` only ever runs the specified commit.

The defense is mechanical: pin to a commit. The commit is content-addressable and immutable; the tag is a movable label that depends on the action's maintainer staying both honest and uncompromised.

## How to do it

For every step that references a third-party action, replace the tag or branch with a full 40-character commit SHA, and keep a comment naming the human-readable version it represents:

```yaml
steps:
  - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v5.0.0
  - uses: actions/setup-node@2028fbc5c25fe9cf00d9f06a71cc4710d4507903  # v6.0.0
  - uses: ossf/scorecard-action@62b2cac7ed8198b15735ed49ab1e5cf35480ba46  # v2.4.0
```

Three references that look like they need pinning but don't:

- Local actions (`uses: ./` or `uses: ../`) — they version with the workflow's own commit. Leave them as paths.
- GitHub-published reusable workflows you fully trust at the org level (`uses: <your-org>/<repo>/.github/workflows/foo.yml@<sha>`) — pinning is still preferred, but tag references inside your own org carry the same trust as the rest of your code.
- SLSA Build L3 generator workflows (`slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.1.0`) — these require a tag reference for the SLSA verifier to validate the workflow's signed release. Allowlist them explicitly in `.deterministic-deps.yml` (see [Common pitfalls](#common-pitfalls) below).

### Finding the right SHA

For a lightweight tag, the API gives you the commit directly:

```sh
gh api repos/actions/checkout/git/ref/tags/v5.0.0 -q '.object.sha'
```

For an annotated tag (created with `git tag -a`), the first call returns the tag object's SHA; dereference one more level to get the commit:

```sh
TAG_SHA=$(gh api repos/<org>/<repo>/git/ref/tags/<tag> -q '.object.sha')
COMMIT_SHA=$(gh api repos/<org>/<repo>/git/tags/${TAG_SHA} -q '.object.sha')
echo "${COMMIT_SHA}"
```

You can tell which is which from the `.object.type` field: `commit` is the lightweight case; `tag` means annotated.

### Tooling that automates pinning and bumps

- [**`pin-github-action`**](https://github.com/mheap/pin-github-action) — npm CLI that rewrites all `uses:` references in a workflow to SHA pins with the version comment alongside. Useful for a one-shot conversion of an existing repo.
- **Dependabot's `github-actions` ecosystem** — once your actions are SHA-pinned, Dependabot opens a PR each time the action's release tag advances, with the new SHA already inserted and the version comment refreshed. This is the maintenance answer. See the `.github/dependabot.yml` in this repo for a working config.
- **Renovate** — supports `pinDigests: true` for the same behavior. Choose Renovate or Dependabot, not both.
- [**`step-security/secure-repo`**](https://github.com/step-security/secure-repo) — a browser-based tool that rewrites a workflow file in place with SHA pins and other Tier 1/2 hardening.

## How to verify

- **`deterministic-deps` rules** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `github-actions/sha-pin` (high) — external `uses:` references must include a full 40-character commit SHA
  - `github-actions/full-sha` (high) — short SHAs (anything fewer than 40 hex characters) are flagged
  - `github-actions/docker-digest` (high) — `uses: docker://<image>` references must include `@sha256:<digest>`
- **Alternatives:** [OpenSSF Scorecard's `pinned-dependencies` check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) (covers GitHub Actions and Dockerfile pins); `pin-github-action --check` runs in dry-run mode and exits non-zero on any tag reference; `step-security/secure-repo` reports the same as part of its score.
- **Manual one-liner:**

  ```sh
  # Flag any external `uses:` line that doesn't end in a 40-char hex SHA.
  grep -rEn '^\s*-?\s*uses:\s*[^./@][^@]*@[^[:space:]]+' .github/workflows/ \
    | grep -vE '@[0-9a-f]{40}(\s|$|#)' \
    && echo "tag or short-SHA reference found"
  ```

## Common pitfalls

1. **Pinning to an annotated tag object's SHA instead of the underlying commit.** Both work at runtime, but downstream supply-chain tools and audits expect the commit SHA. Use the dereferencing snippet above; the [`deterministic-deps` config](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/configuration.md) treats either form as valid, but consistency wins.
2. **Pinning local actions.** `uses: ./.github/actions/my-action` and `uses: ../shared-action` are versioned by the workflow's own checkout — they are already deterministic. Leave them alone.
3. **Pinning a SLSA reusable workflow.** SLSA Build L3 verification requires the workflow be referenced by a signed release tag, not a SHA. Add an allowlist entry in `.deterministic-deps.yml`:

   ```yaml
   allowlist:
     - file: .github/workflows/release.yml
       ruleId: github-actions/sha-pin
   ```

   Other reusable workflows that require tag references for trust verification (some Step Security ones, some `actions/` first-party reusables) get the same treatment. Keep the allowlist narrow — one entry per file, per ruleId.
4. **Forgetting Docker action references.** `uses: docker://image:tag` is a third-party reference too; pin to `@sha256:<digest>`. The same goes for container-action `image:` fields inside an `action.yml`.
5. **Not maintaining a renewal cadence.** SHA pins go stale. Dependabot or Renovate keep them flowing automatically. The Tier 1 security win is having known inputs at build time, not having unchanged inputs forever — the goal is bounded, deliberate updates, not stasis.

## Real example

[`Ozark-Security-Labs/deterministic-deps/.github/workflows/ci.yml`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/590b60e05138c3a56918f66078cb8759646c7cae/.github/workflows/ci.yml) — every external `uses:` reference is a 40-character commit SHA. The repo's [`.github/dependabot.yml`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/590b60e05138c3a56918f66078cb8759646c7cae/.github/dependabot.yml) keeps those SHAs flowing.
