# Rule 3.7 — Protect the default branch and release tags with rulesets

> Tier 3 · Production

## Rule

The default branch and the `v*` release tag namespace are protected by GitHub [Repository Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets). Direct pushes blocked, fast-forwards required, signed commits required, at least one approving review required, status checks required to pass, tag retagging blocked. The ruleset JSON is committed to the repo under `.github/rulesets/` so it's reviewable in source.

## Why it matters

Every other Tier 1–3 pattern enforces something *during* a CI run or *at* release time. None of them stop a maintainer (or a compromised maintainer account) from pushing a malicious commit straight to `main`, retagging a published `v1.2.3`, or deleting a branch to hide history. The platform-level controls that block those actions live in branch protection.

For supply chain integrity, the tag-protection piece is especially load-bearing: SLSA provenance (Rule 3.1) and cosign signatures (Rule 3.3) are bound to the tag the release was built from. If the tag can be moved, the verification chain becomes circular — "the artifact matches the tag" is meaningless when "the tag" is whatever a write-access maintainer last pointed it at. Tag protection makes the binding immutable.

Modern rulesets (the JSON-defined replacement for the legacy branch-protection-rules UI) live alongside the code. That brings two benefits: they're reviewable in PRs (changes to security posture go through the same review the code does), and they're portable (a new repo can adopt the same posture by copying the JSON).

## How to do it

Create `.github/rulesets/main-protection.json`:

```json
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_linear_history" },
    { "type": "required_signatures" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true,
        "allowed_merge_methods": ["squash", "rebase"]
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "do_not_enforce_on_create": false,
        "required_status_checks": [
          { "context": "dependency determinism" },
          { "context": "docs quality" }
        ]
      }
    }
  ],
  "bypass_actors": []
}
```

The rules in plain English:

| Rule | Blocks |
| --- | --- |
| `deletion` | `git push --delete origin main` |
| `non_fast_forward` | Force-pushes that rewrite history |
| `required_linear_history` | Merge commits (forces squash or rebase) |
| `required_signatures` | Unsigned commits (GPG / SSH signature required) |
| `pull_request` | Direct pushes; requires a PR with ≥1 approval, code-owner review, no stale reviews, all comments resolved |
| `required_status_checks` | Merging before named CI checks pass |

`bypass_actors: []` means *no one* bypasses, including admins. Tighten as needed; the default OSL stance is empty.

### Apply the ruleset to GitHub

The JSON file is the source of truth; applying it to the repo settings is a one-time step (and re-applied when the file changes):

```sh
gh api -X POST /repos/your-org/your-repo/rulesets \
  --input .github/rulesets/main-protection.json
```

For an existing ruleset, use `PUT /repos/.../rulesets/<id>` to update. A small `scripts/apply-rulesets.sh` that diffs current state against the JSON and applies changes is a sensible follow-up.

### Tag protection ruleset

A separate ruleset for the `v*` tag namespace:

```json
{
  "name": "Protect release tags",
  "target": "tag",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/tags/v*"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "update" }
  ],
  "bypass_actors": []
}
```

The `update` rule blocks moving an existing tag. Combined with `deletion`, this means a `v1.2.3` tag, once pushed, cannot be moved or removed — exactly the immutability SLSA and cosign verification depends on.

SessionScope's release workflow validates that this exact ruleset is active before proceeding ([release.yml line ~25](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/release.yml)): if the ruleset is missing or inactive, the release fails fast rather than producing artifacts whose tag can be retroactively moved.

### Required signatures (commit signing)

`required_signatures: true` blocks commits without GPG or SSH signatures. Contributors need to set up signing once:

```sh
# SSH-based commit signing (recommended for new setups; uses the same key as gh auth)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

GitHub verifies the signature against the user's published keys at github.com/<user>.keys. The "Verified" badge in commit history is the signal that signing is working.

## How to verify

- **`deterministic-deps`** doesn't check rulesets — it's static-analysis of in-tree files, not GitHub-API state.
- **Alternatives:**
  - The [`AlexJReid/branch-protection-bot`](https://github.com/marketplace/actions/branch-protection-bot) action applies branch-protection JSON in CI; community-maintained ruleset-drift-detection bots exist but none has emerged as canonical yet
  - GitHub's own [Repository Settings API](https://docs.github.com/en/rest/repos/rules) is the ground truth; any inspection script that uses `gh api .../rulesets` is canonical
- **Manual:**

  ```sh
  # List active rulesets
  gh api /repos/your-org/your-repo/rulesets --jq '.[] | {name, target, enforcement, id}'

  # Inspect one
  gh api /repos/your-org/your-repo/rulesets/<id> | jq .
  ```

## Common pitfalls

1. **Setting rulesets in the GitHub UI and not committing the JSON.** The settings are now reviewable only in the UI, drift becomes invisible, and onboarding a new repo means clicking through screens instead of applying a template. Always commit the JSON.
2. **`bypass_actors` that includes "Maintain" or higher.** That's almost everyone. A bypass meaningfully exists for emergency response — keep it empty in steady-state, populate it via temporary ruleset during incidents.
3. **`required_status_checks` listing checks that don't exist.** GitHub silently treats them as pending forever, blocking all PRs. Verify check names match exactly what GitHub Actions emits.
4. **Forgetting tag-protection.** Most OSS repos have branch protection but not tag rulesets — and tag rulesets are what makes SLSA / cosign / `npm --provenance` actually trustworthy. Tier 3 isn't complete without them.
5. **Treating `required_signatures` as optional.** Without signed commits, the `required_signatures` rule blocks everyone; with them, only contributors who haven't set up signing are blocked, which surfaces the gap immediately and is the desired behaviour.

## Real example

[`Ozark-Security-Labs/PkgWarden/.github/rulesets/main-protection.json`](https://github.com/Ozark-Security-Labs/PkgWarden/blob/d4db739a5fb6c1a06fa034c6e9b365cf6e141135/.github/rulesets/main-protection.json) is the canonical OSL ruleset: deletion blocked, non-fast-forward blocked, linear history required, signatures required, PR-with-1-approving-review-and-codeowner-review required, single status check (`repository hygiene`) required. The companion tag-protection ruleset is referenced by [SessionScope's release.yml](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/release.yml) which verifies it exists with the name `Protect release tags` before any release artifact is produced.

The tag ruleset JSON itself is not currently committed to a public OSL repo (it's applied via the GitHub UI on SessionScope and AuthMap); committing it under `.github/rulesets/release-tag-protection.json` alongside `main-protection.json` is a near-term follow-up.
