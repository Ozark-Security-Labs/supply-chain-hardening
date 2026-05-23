# Rule 2.5 — Use Dependabot or Renovate with grouped updates

> Tier 2 · Hardened

## Rule

Pinned dependencies (Tier 1) plus active enforcement (Rules 2.1, 2.4) without an automated update flow becomes stasis — pins age, advisories pile up, and the next bulk upgrade is a one-week tax. Configure Dependabot or Renovate with **grouped** PRs so the team gets a manageable cadence of bumps without a PR per dependency per week.

## Why it matters

Every dependency pin you make in Tiers 1 and 2 only secures the version you pinned _at_. Patch releases ship security fixes; new minor versions ship the security fixes that won't be backported. The point of pinning is bounded, deliberate updates — not "never change anything again." A repo with current pins and an active bot is materially safer than a repo with old pins and no bot; the former gets fixes for advisories that ship after the pin date, the latter doesn't.

Grouping matters because the alternative — one PR per dependency, sometimes 10–20 per week on a busy project — drives PR fatigue. The team stops reviewing them carefully, starts auto-merging on faith, and the supply-chain hardening you built becomes ceremonial. Grouping bundles routine bumps into one PR per ecosystem per cadence, which is reviewable.

## How to do it

### Dependabot (recommended for GitHub-native projects)

A `.github/dependabot.yml` with grouped updates per ecosystem:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:30"
      timezone: America/Chicago
    open-pull-requests-limit: 5
    labels:
      - dependencies
      - github-actions
    groups:
      production-dependencies:
        applies-to: version-updates
        update-types:
          - patch
          - minor
      security-updates:
        applies-to: security-updates

  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
      day: monday
      time: "09:45"
      timezone: America/Chicago
    open-pull-requests-limit: 5
    labels:
      - dependencies
      - npm
    groups:
      production-dependencies:
        dependency-type: production
        update-types:
          - patch
          - minor
      development-dependencies:
        dependency-type: development
        update-types:
          - patch
          - minor
      security-updates:
        applies-to: security-updates
```

Key choices:

- **Separate groups for `security-updates`.** Dependabot opens these as soon as an advisory drops, outside the weekly cadence. Don't bundle them with routine bumps.
- **Patch + minor together; major separate.** Major version bumps need human review and are not generally safe to bundle. Dependabot keeps them as their own PRs by default.
- **One ecosystem per `updates:` entry.** Dependabot doesn't bundle across ecosystems (you can't group an `npm` patch with a `github-actions` patch).
- **`open-pull-requests-limit: 5`** caps the total open Dependabot PRs per ecosystem so a stale repo doesn't suddenly emit fifty PRs when re-enabled.

### Renovate (richer config, multi-platform, semi-self-hosted)

The equivalent `renovate.json` at the repo root:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "schedule": ["before 10am on monday"],
  "timezone": "America/Chicago",
  "labels": ["dependencies"],
  "prConcurrentLimit": 5,
  "packageRules": [
    {
      "groupName": "patch and minor",
      "matchUpdateTypes": ["patch", "minor"]
    },
    {
      "groupName": "github-actions",
      "matchManagers": ["github-actions"]
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false
    }
  ],
  "vulnerabilityAlerts": {
    "labels": ["security"],
    "automerge": false
  },
  "pinDigests": true
}
```

`pinDigests: true` is the Tier 1 + Tier 2 enforcement: Renovate will refuse to leave tag references alone and will replace them with SHA pins, the same as Rule 1.2 and Rule 2.3.

### Pick one

Dependabot or Renovate, not both. They will fight each other over the same dependency, each opening competing PRs. Renovate is more configurable and supports more ecosystems (Bitbucket, GitLab, self-hosted); Dependabot is GitHub-native, requires no config beyond the `.yml`, and integrates with GitHub Security advisories out of the box. Most public OSS projects on GitHub use Dependabot for that reason.

### Auto-merge cautiously

Both tools support auto-merge of grouped patch PRs once CI is green. Use it sparingly: auto-merging is auto-trust of the dependency bump. It only makes sense when:

- The repo has the full Tier 1 + Tier 2 enforcement (so a malicious bump that violated determinism would fail CI)
- The repo has Tier 3 release attestations (so a downstream consumer can re-verify what was built)
- The team accepts that a hostile maintainer's patch can land without human review

The conservative position is: never auto-merge; treat the weekly Dependabot/Renovate PR as a "review one thing per week" cost.

## How to verify

- **`deterministic-deps`** doesn't cover update cadence — it scans the current state, not the history. Auditing whether updates are flowing is observational, not static.
- **Alternatives:** [Snyk](https://snyk.io/), [Sonatype Lift](https://www.sonatype.com/), and the self-hosted [`renovatebot/renovate`](https://github.com/renovatebot/renovate) container provide similar bots with extra reporting layers; the underlying mechanic is the same.
- **Manual:** check the repo's PR list filtered by `is:pr author:app/dependabot` (or `author:app/renovate`) at least monthly. If there are no PRs in the last 30 days, either nothing has needed updating (rare) or the bot isn't running (likely — check the workflow run history at `actions`).

## Common pitfalls

1. **Wiring both Dependabot and Renovate.** They overlap. You get two PRs for every bump and one of them probably never closes cleanly.
2. **Setting `open-pull-requests-limit: 1`** to "minimize noise." It throttles security updates too. Use `5` for routine plus `dependency-type: security` separately.
3. **Grouping major and patch updates together.** A grouped PR that includes a major bump is impossible to merge cleanly without rolling the major back. Keep majors as their own PR by default.
4. **Auto-merging without Tier 1 + 2 + 3 enforcement.** Auto-merge presupposes that CI catches a hostile bump. If your CI doesn't run `deterministic-deps`, dependency-review, AND CodeQL on every PR, you're trusting Dependabot/Renovate to vet things they don't vet.
5. **Forgetting `pinDigests: true` in Renovate.** Renovate happily bumps tag-referenced actions to a new tag — not a SHA. Without `pinDigests: true`, you undo Rule 1.2 every week.

## Real example

[`Ozark-Security-Labs/forkguard/.github/dependabot.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.github/dependabot.yml) wires Dependabot for both `github-actions` and `gomod` ecosystems with weekly Monday-morning Central-time PRs. Honest gap: the forkguard config does not yet use `groups:` — it emits one PR per bump — so Tier 2's grouped pattern is illustrated by the inline snippet above rather than by the linked example. Adding `groups:` to the OSL repos is a follow-up.
