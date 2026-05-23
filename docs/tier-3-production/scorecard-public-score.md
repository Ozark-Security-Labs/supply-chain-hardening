# Rule 3.6 — Publish an OpenSSF Scorecard score (and act on the findings)

> Tier 3 · Production

## Rule

Run [OpenSSF Scorecard](https://scorecard.dev/) on a schedule, publish the results to scorecard.dev, badge the README with the current score, and keep the score above 7. The score communicates posture; the published findings communicate where the work remains.

## Why it matters

Tiers 1 and 2 add specific hardening patterns. The aggregate question — "what's the overall security posture of this project?" — is hard for a downstream consumer to answer by reading individual rule outcomes. Scorecard is the OSS-wide standard for that aggregate: a numeric 0–10 score, 18 individual checks, and a public dashboard at scorecard.dev that any consumer can compare across projects.

For supply-chain hygiene specifically, Scorecard's most relevant checks overlap directly with Tiers 1 and 2: `Pinned-Dependencies`, `Token-Permissions`, `Branch-Protection`, `Code-Review`, `Dangerous-Workflow`, `Dependency-Update-Tool`, `SAST`. A repo that has cleanly adopted Tiers 1 and 2 typically scores 7–9 on Scorecard without further intervention; the remaining gap is usually Tier 3 items like `Signed-Releases` and `Packaging`.

Publishing the score makes it auditable. A downstream user who needs to vouch for the project's posture to their security team can point at the badge instead of paraphrasing.

## How to do it

A workflow at `.github/workflows/scorecard.yml`:

```yaml
name: scorecard

on:
  branch_protection_rule:
  schedule:
    - cron: "0 6 * * 1"      # weekly, Monday early-morning UTC
  push:
    branches:
      - main

permissions: read-all

jobs:
  analysis:
    name: scorecard analysis
    runs-on: ubuntu-24.04
    permissions:
      security-events: write    # SARIF upload to Code Scanning
      id-token: write           # OIDC for publish_results
      contents: read
      actions: read
    steps:
      - uses: actions/checkout@<sha>
        with:
          persist-credentials: false

      - uses: ossf/scorecard-action@<sha>
        with:
          results_file: results.sarif
          results_format: sarif
          publish_results: true     # publish to scorecard.dev

      - uses: actions/upload-artifact@<sha>
        with:
          name: scorecard-sarif
          path: results.sarif
          retention-days: 5

      - uses: github/codeql-action/upload-sarif@<sha>
        with:
          sarif_file: results.sarif
```

Three things matter:

**`publish_results: true`** is the gate that puts your score on scorecard.dev. Without it, the score is private to your repo's Code Scanning UI. The OSS norm is to publish; the security gain of *not* publishing is minimal (the SARIF includes the same data) and you lose the public signalling.

**SARIF upload to Code Scanning.** Scorecard findings land alongside `deterministic-deps` (Rule 2.1), CodeQL (Rule 2.7), and any other SARIF-emitting check. One inbox for triage.

**The triggers matter.** `branch_protection_rule` re-runs Scorecard whenever you change branch protection settings (so the score updates immediately). `schedule` keeps it fresh. `push: branches: [main]` runs on every merge so the dashboard reflects the current state.

### Add the badge to your README

```markdown
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/your-org/your-repo/badge)](https://scorecard.dev/viewer/?uri=github.com/your-org/your-repo)
```

The badge image URL is stable; the viewer URL takes you to the breakdown of the score with each check explained. Put it near the top of the README, in the same row as the build status and license badges.

### Reading and acting on findings

The 18 checks are documented at [`ossf/scorecard/docs/checks.md`](https://github.com/ossf/scorecard/blob/main/docs/checks.md). The ones most affected by Tiers 1–2:

| Check | What raises the score |
| --- | --- |
| `Pinned-Dependencies` | Tier 1.2 (SHA-pinned actions) + Tier 2.3 (container digests) |
| `Token-Permissions` | Tier 1.5 (minimal workflow permissions) |
| `Branch-Protection` | Tier 3.7 (rulesets) |
| `Code-Review` | Tier 3.7 (required reviewers) |
| `Dangerous-Workflow` | Generally absent if Tier 2's hygiene is in place |
| `Dependency-Update-Tool` | Tier 2.5 (Dependabot or Renovate) |
| `SAST` | Tier 2.7 (CodeQL) |
| `Signed-Releases` | Tier 3.3 (cosign) — and Tier 3.1 (SLSA) partially satisfies it |
| `Packaging` | Tier 3.5 (registry provenance) |
| `Webhooks` | Outside the supply-chain hardening scope |

A repo on the full Tier 1 + Tier 2 + Tier 3 stack typically scores 8–10. Below 7 generally means a missing Tier 1 or 2 pattern; below 5 means major gaps. Treat the score as a posture signal, not a target — but a score that drops month-over-month is a useful regression alert.

### When you can't fix a check

Some checks (e.g., `Webhooks`, `License`) depend on org-level settings outside the workflow's reach. Document the exclusion in your README (or in a `.scorecard.yml` config) so the badge reflects what you can control. Scorecard's design is "best-effort signal," not "audit-grade."

## How to verify

- **`deterministic-deps`** doesn't replicate Scorecard — Scorecard is the meta-check. They cover different surfaces; both belong on every repo.
- **Alternatives:**
  - [Snyk Open Source](https://snyk.io/product/open-source-security-management/) — proprietary, broader risk surface
  - [Socket](https://socket.dev/) — supply-chain-specific, including npm-package behaviour analysis
  - The [OpenSSF Best Practices Badge](https://www.bestpractices.dev/) is a self-attested questionnaire (rather than automated check); complementary to Scorecard
- **Manual:** `scorecard --repo=github.com/your-org/your-repo` (after installing the CLI) reproduces the workflow output locally. Useful for debugging a check that's failing in CI but you don't understand why.

## Common pitfalls

1. **Setting `publish_results: false` to "keep findings private."** The SARIF goes to Code Scanning regardless; only the public *score* is hidden. The privacy gain is illusory, the signal loss is real.
2. **Badging a score that's months out of date.** The badge image is fetched at render time from scorecard.dev; if your workflow hasn't run recently, the API serves the stale score. Make sure the weekly schedule actually runs.
3. **Reacting to a single low-severity check by adding an exception comment instead of fixing it.** Most Scorecard checks have a clear remediation in the breakdown. Fixes are usually small (e.g., `Token-Permissions` is one `permissions:` block).
4. **Treating a 10/10 score as a security guarantee.** It isn't. Scorecard is a structural signal — none of the checks ensure the code is *correct*; they just ensure the project structure makes correctness easier to verify.
5. **Forgetting the OIDC `id-token: write` permission** for `publish_results: true`. Without it, the publish step fails silently (the score still uploads to Code Scanning, but doesn't appear on scorecard.dev).

## Real example

[`Ozark-Security-Labs/PkgWarden/.github/workflows/scorecard.yml`](https://github.com/Ozark-Security-Labs/PkgWarden/blob/d4db739a5fb6c1a06fa034c6e9b365cf6e141135/.github/workflows/scorecard.yml) is the standard pattern in the OSL portfolio: weekly schedule, `branch_protection_rule` trigger, `publish_results: true`, SARIF to Code Scanning. The supply-chain-hardening guide itself (this repo's [`scorecard.yml`](https://github.com/Ozark-Security-Labs/supply-chain-hardening/blob/main/.github/workflows/scorecard.yml)) dogfoods the same template.
