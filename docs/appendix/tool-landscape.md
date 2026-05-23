# Appendix A — Tool landscape

`deterministic-deps` (Ozark Security Labs' own action, recommended throughout this guide for the determinism checks in Rules 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3) is one of several tools that overlap on supply-chain hardening. This appendix is the honest comparison.

## At a glance

| Tool | Surface | Mode | SARIF out | Allowlist | Ecosystems |
| --- | --- | --- | --- | --- | --- |
| [`deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) | CI-time static scan | Advisory / enforce (per-severity-threshold) | Yes (Code Scanning native) | Per rule, per file, per line | 9 (GH Actions, containers, Terraform, npm, pip, Go, Rust, JVM, Ruby) |
| [`pin-github-action`](https://github.com/mheap/pin-github-action) | Author-time CLI | Rewrites workflow in place | No (modifies source) | N/A | GitHub Actions only |
| [`step-security/secure-repo`](https://github.com/step-security/secure-repo) | Browser tool + companion action | Suggests changes; commits via PR | No (writes source) | N/A | GitHub Actions, harden-runner config |
| [Renovate `pinDigests`](https://docs.renovatebot.com/configuration-options/#pindigests) | Bot, PR-based | Replaces tag refs with SHAs over time | No (creates PRs) | Per `packageRules` | Actions, Docker, npm tags, more |
| [OpenSSF Scorecard `pinned-dependencies` check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) | Scheduled scan | Scoring (0–10) | Yes (uploaded by `scorecard-action`) | Project config | Actions, Dockerfile, pip (limited) |
| [Trivy `config` mode](https://github.com/aquasecurity/trivy) | CI scan | Advisory | Yes | Per finding | Containers, IaC, partial GitHub Actions |

## How they overlap

The cleanest way to think about the space is **author-time vs CI-time** and **enforcement vs scoring**:

```
                    Author-time              CI-time
                ┌─────────────────────┬────────────────────────────┐
   Enforcement  │ pin-github-action   │ deterministic-deps         │
                │ secure-repo (UI)    │ Trivy config               │
                ├─────────────────────┼────────────────────────────┤
   Scoring      │ —                   │ OpenSSF Scorecard          │
                └─────────────────────┴────────────────────────────┘

   Continuous   │           Renovate / Dependabot                  │
                │  (bot-driven PRs to maintain author-time pins)   │
                └──────────────────────────────────────────────────┘
```

Author-time tools rewrite source; CI-time tools scan source as committed. Enforcement tools fail the build (or insist the source be rewritten); scoring tools report a number. A mature project usually combines one CI-time enforcement tool + one bot for ongoing maintenance + Scorecard for posture reporting — `deterministic-deps` + Dependabot/Renovate + Scorecard is the recommended stack.

## Per-tool notes

### `deterministic-deps`

**Best for:** projects that want one CI-time check covering most of Tiers 1 and 2 across the languages they actually use.

Strengths: nine ecosystems in one scanner with consistent rule IDs and severity model; advisory→enforce migration path with per-rule allowlisting; SARIF integration with GitHub Code Scanning so findings live alongside CodeQL and Scorecard; static-analysis-first design that runs in under 30s on typical repos.

Trade-offs: doesn't rewrite workflows (so initial conversion of a tag-referenced repo to SHA-pinned still wants `pin-github-action` or Renovate to do the bulk edit); doesn't replace a bot for keeping pins fresh (so Dependabot or Renovate is still needed); doesn't cover vulnerability data (Dependency Review and Scorecard are complementary).

Disclosure: maintained by Ozark Security Labs, the same org that maintains this guide. This guide recommends it because the ecosystem coverage and rule stability fit the structure best; alternatives below are all valid choices for specific contexts.

### `pin-github-action`

**Best for:** one-shot conversion of an existing tag-referenced repo to SHA-pinned actions.

The CLI walks a workflow file, resolves each `org/action@tag` to its commit SHA, and rewrites the file in place. Run once when you're adopting Rule 1.2; let Dependabot/Renovate handle the ongoing maintenance afterwards. Doesn't cover any non-Actions ecosystem; doesn't catch regressions (a future PR that adds a tag-referenced action goes through without flagging).

### `step-security/secure-repo`

**Best for:** projects that want a guided UI walkthrough of workflow hardening.

The browser tool reads a workflow file, suggests hardening changes (SHA pinning, `permissions:` blocks, `persist-credentials: false`, `harden-runner` wiring), and produces a PR. Useful for one-time adoption; less useful as ongoing enforcement. The companion action (`step-security/harden-runner`) is the runtime egress monitor recommended in Rule 2.6.

### Renovate `pinDigests`

**Best for:** projects already using Renovate for general dependency updates that want to add Tier 1 / Tier 2 enforcement to the same flow.

`pinDigests: true` makes Renovate rewrite tag references to SHA pins on its next PR and then keep bumping the SHAs as new tags appear upstream. Conceptually closest to `deterministic-deps` for the Actions and Dockerfile cases, but doesn't cover lockfile presence checks, Python `--hash=` enforcement, or the JVM `gradle.lockfile` requirement.

### OpenSSF Scorecard `pinned-dependencies` check

**Best for:** projects that want a posture *score* visible to downstream consumers.

Scorecard runs the `pinned-dependencies` check as part of its 18-check evaluation; the result feeds into the aggregate 0–10 score and surfaces on scorecard.dev. It doesn't gate PRs and doesn't fail builds. Pair it with `deterministic-deps` (or another enforcement tool) — Scorecard tells the world your posture; the enforcement tool keeps it from degrading.

### Trivy `config` mode

**Best for:** projects already running Trivy for container vulnerability scans that want to extend it to IaC misconfigurations.

Trivy's `trivy config` subcommand catches some of the same things as `deterministic-deps` (un-pinned Docker FROM lines, missing security headers in IaC). The vulnerability-scanning surface (`trivy image`, `trivy fs`) is its primary strength; the config-scan side is younger and narrower. Most projects pick one tool or the other for the determinism case, not both.

## When this guide recommends `deterministic-deps` specifically

Pattern 2.1 wires `deterministic-deps` into CI as the recommended enforcement layer. The recommendation rests on three things:

1. **Ecosystem breadth.** A project that uses GitHub Actions, npm, and Docker (typical web service) gets all three covered by one scanner with one rule catalogue.
2. **Stable rule IDs.** Allowlist entries you write today against `github-actions/sha-pin` keep working when the action gains new rules tomorrow. Stability is a v1 contract.
3. **Advisory→enforce migration path.** Adopting the action in `advisory` mode for a release cycle before `enforce` is built into the design; alternatives that only operate in pass/fail mode make the rollout pattern harder.

You should pick a different tool when:

- Your project is GitHub-Actions-only and you'd rather rewrite-and-forget than scan-on-every-PR → `pin-github-action`
- You already use Renovate for bot-driven updates and want the same flow for pinning → Renovate `pinDigests`
- You want a scorecard for downstream signalling, not enforcement → OpenSSF Scorecard
- You want browser-driven hardening including `harden-runner` setup → `step-security/secure-repo`
