# Appendix C — Migration playbook

Adopting these patterns on a brand-new project is straightforward — the relevant work fits in the initial repo scaffold. Retrofitting them onto a project that already has contributors, a release cadence, and a deployed user base without burning out the team is the actual hard problem. This appendix is the tested order of operations.

## Principles

Three things to internalise before starting:

1. **Adopt one tier at a time.** A PR that tries to land Tiers 1, 2, and 3 simultaneously is unreviewable and breaks too much at once. Treat each tier as its own multi-PR workstream.
2. **Advisory mode for at least one release cycle before enforce.** Every enforcement gate this guide recommends (Pattern 2.1 `deterministic-deps`, Pattern 2.6 `harden-runner`) supports a non-blocking mode. Use it long enough for the team to see what would break before it starts breaking.
3. **Communicate the why before the what.** Tier 2 and Tier 3 patterns add CI checks and process gates; contributors who don't understand the threat model will see them as bureaucracy. A 15-minute team brief on the [threat model](../#the-threat-model) before the first enforcement PR pays for itself.

## Order of operations

### Phase 1 — Tier 1 (about a week of elapsed time, ~4 hours of work)

Tier 1 is mechanical. Land it in one PR per major change so the diff stays reviewable:

1. **PR A**: Commit the lockfile (Rule 1.1) for each package manager in the repo. Convert any CI `npm install` → `npm ci`, `cargo build` → `cargo build --locked`, etc.
2. **PR B**: SHA-pin every third-party Action (Rule 1.2). Use `pin-github-action` or `step-security/secure-repo` to do the bulk conversion; review the resulting diff for any actions you don't recognise.
3. **PR C**: Pin runner OS (Rule 1.3), pin language toolchain (Rule 1.4), add minimal `permissions:` blocks (Rule 1.5). These three are individually small; bundling them saves review rounds.

After Phase 1, your repo *can* be deterministic. Nothing fails when a regression slips in yet; that's Phase 2.

### Phase 2 — Tier 2 (one to two release cycles of elapsed time)

Tier 2 adds the enforcement layer. The order matters because some checks depend on others:

1. **Week 1 — Wire `deterministic-deps` in advisory mode** (Rule 2.1). Don't change anything else. Let the workflow run on every PR and `main` push for at least one release cycle. The findings list is your Tier 1 gap inventory.
2. **Week 2 — Close the gaps the advisory output found.** Each fix is small; the cumulative effect is that the next `enforce` flip is safe.
3. **Week 3 — Promote to enforce** (Rule 2.1, last step). At the same PR, add the rules that require enforcement to be meaningful: Dependency Review (Rule 2.4), CodeQL (Rule 2.7).
4. **Week 4 — Bot-driven updates** (Rule 2.5). Configure Dependabot or Renovate with grouping; this is the long-term sustainability play.
5. **Later** — `persist-credentials: false` + harden-runner (Rule 2.6). Harden-runner specifically wants its own advisory→block migration; budget one release cycle for the egress allowlist to stabilise.
6. **Ecosystem-specific gaps**: Hash-pin Python requirements (Rule 2.2) and container image digests (Rule 2.3) if your project uses Python or Dockerfiles.

After Phase 2, regressions in Tier 1 patterns fail PRs automatically and your dependency surface has active vulnerability and license checks.

### Phase 3 — Tier 3 (a focused two-week effort or a longer background workstream)

Tier 3 changes your release process. Don't pipeline it with Tiers 1 and 2 — it's a separate enough surface that mixing the two workstreams confuses everyone.

1. **Spike on a non-production tag first.** Cut a `v0.0.0-rc1` tag against a `release-pipeline-v2` branch, watch the SLSA + SBOM + signing flow run end-to-end. Iterate on the workflow until it produces what you want; *then* promote the workflow to `main` and tag the real release.
2. **Branch & tag protection rulesets (Rule 3.7) go in early.** They block bad releases before you've automated the good releases. Commit the JSON, apply via `gh api`, document the bypass conditions.
3. **SLSA provenance + SBOM + Scorecard (Rules 3.1, 3.2, 3.6) are the next layer.** Wire them into the existing release workflow.
4. **OIDC for cloud / registry pushes (Rule 3.4) replaces secrets one secret at a time.** Migrate the lowest-blast-radius one first to learn the trust-policy mechanics; keep going until the secret list is empty.
5. **Cosign signing (Rule 3.3) and registry-native provenance (Rule 3.5) come last.** They're useful but not strictly required to claim Tier 3 — SLSA + SBOM + reproducibility covers most of the same trust surface. Add them when the rest is stable.

## What to do when a PR fails because of new enforcement

The first PR after each enforcement flip will fail for some contributor. Three responses, in order:

1. **Read the finding and fix it in the same PR.** Usually a one-line change (rename `npm install` to `npm ci`, switch a tag reference to a SHA, add a `permissions:` block). Default response.
2. **Add an allowlist entry** if the finding is legitimately a non-issue (SLSA reusable workflow tag references are the canonical case). Reference the relevant pattern doc in the allowlist comment so future reviewers know why.
3. **Open an issue to revisit the rule**. Rare; reserved for cases where the rule is genuinely wrong for your context.

Avoid the fourth option: disabling the rule globally to make the PR pass. That undoes the work.

## When to back off

If contributors are openly frustrated, your enforcement is moving faster than the team can absorb. Drop back to advisory for the offending rule, fix the gap on a separate workstream, then promote back to enforce. The defensive value of Tier 2 only materialises when the team understands and accepts the checks; an enforced rule that everyone routinely bypasses is worse than an advisory rule that gets acted on.

## Time budget

A rough per-tier time-to-adoption for a mid-sized OSS project with active maintainers and a weekly-ish release cadence:

| Phase | Calendar time | Engineer time |
| --- | --- | --- |
| Tier 1 (Phase 1) | 1 week | ~4 hours |
| Tier 2 (Phase 2) | 4–6 weeks | ~2 hours per week (mostly waiting for advisory output, then a focused half-day per enforcement flip) |
| Tier 3 (Phase 3) | 2–4 weeks of focused effort, or a longer background workstream | 2–4 days of release-flow rewriting + iteration |

Faster is possible; slower is also fine. The goal is the steady state, not the calendar.
