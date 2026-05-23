# Rule 2.1 — Wire `deterministic-deps` into CI

> Tier 2 · Hardened

## Rule

Run [`deterministic-deps`](https://github.com/Ozark-Security-Labs/deterministic-deps) on every PR and push. Start in advisory mode for one or two release cycles, then flip to enforce at the lowest severity threshold so any new non-deterministic dependency reference fails the build.

> **Disclosure:** `deterministic-deps` is maintained by Ozark Security Labs, the same group that maintains this guide. The honest comparison to alternatives lives in the [Tool Landscape](../appendix/tool-landscape.md) appendix; you may prefer a different tool. This pattern documents the recommended one because we know its rule catalogue best, not because there is no other defensible choice.

## Why it matters

Tier 1 makes a repo capable of being deterministic — lockfiles committed, actions SHA-pinned, runners and toolchain pinned. Nothing in Tier 1 catches the next PR that adds `uses: some-action@v3` to a workflow, or removes `package-lock.json` "to fix a merge conflict," or sneaks in a transitive Python dep without `--hash=`. The defenses degrade silently unless something checks them on every PR.

`deterministic-deps` is the static check. It runs in under 30 seconds on a typical repo, scans nine ecosystems, emits SARIF that lands in GitHub Code Scanning alongside CodeQL findings, and supports per-rule allowlists for the cases where a tag reference is genuinely required (SLSA reusable workflows, for example). The [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md) is the same one cited throughout Tiers 1 and 2 of this guide.

## How to do it

### Advisory rollout (recommended first)

Drop this workflow into `.github/workflows/dependency-determinism.yml`:

```yaml
name: dependency determinism

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "7 10 * * 1"

permissions:
  contents: read
  security-events: write

jobs:
  deterministic-deps:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v5.0.0
      - id: deterministic-deps
        uses: Ozark-Security-Labs/deterministic-deps@<sha>  # v1.1.0
        with:
          mode: advisory
          severity-threshold: low
          sarif: true
      - uses: github/codeql-action/upload-sarif@4f3212b61783c3c68e8309a0f18a699764811cda  # v3.x
        if: always() && steps.deterministic-deps.outputs.sarif-path != ''
        with:
          sarif_file: ${{ steps.deterministic-deps.outputs.sarif-path }}
          category: deterministic-deps
```

Advisory mode posts annotations and uploads SARIF but never fails the build. Use it for one or two release cycles. You will discover three categories of findings:

1. **Real issues** — fix them. This is the value.
2. **Special-case references** that legitimately need a tag (SLSA reusable workflows, certain Step Security reusables) — allowlist them in `.deterministic-deps.yml`.
3. **Confusing findings** that look like false positives — read the rule documentation and the configuration reference before adding an allowlist entry. The rule IDs are stable, so a once-correct allowlist stays correct.

A useful starting `.deterministic-deps.yml`:

```yaml
mode: advisory
severity-threshold: low

exclude:
  - deterministic-deps-report/**
  - reports/**

allowlist: []
```

### Promotion to enforce

When the advisory output has stabilised at zero findings (or stays at a small, known allowlisted set across a release cycle), flip the workflow input and the config file:

```yaml
        with:
          mode: enforce          # was: advisory
          severity-threshold: low
```

```yaml
# .deterministic-deps.yml
mode: enforce
severity-threshold: low
```

The workflow input overrides the config file, but matching them keeps `npx deterministic-deps` runs (local dogfood) consistent with CI. From this point on, any new non-deterministic reference fails the PR check.

The `Fail when deterministic-deps failed` step at the end of the workflow is the gate that propagates the action's exit code; without it, `continue-on-error: true` would mask failures. The full template:

```yaml
      - name: Fail when deterministic-deps failed
        if: steps.deterministic-deps.outcome == 'failure'
        run: exit 1
```

### Remote validation (optional)

`remote-validation: true` makes the action verify that pinned commit SHAs actually exist on GitHub. It is opt-in because it uses the `GITHUB_TOKEN` and depends on the GitHub API being reachable. Recommended once you trust the static checks; surfaces dangling refs (typo'd SHAs, force-deleted commits) that would otherwise show up only at runtime.

## How to verify

- **`deterministic-deps` itself** runs locally to dry-test:

  ```sh
  npx -y Ozark-Security-Labs/deterministic-deps@v1 --mode advisory
  ```

- **Alternatives:** see the [Tool Landscape](../appendix/tool-landscape.md) appendix for `pin-github-action`, Renovate's `pinDigests`, OpenSSF Scorecard's `pinned-dependencies` check, and Step Security's `secure-repo`. None covers the same nine ecosystems, but each is the right answer for some projects.
- **Manual:** review the rendered Code Scanning page after the workflow has run on `main` at least once; findings should be zero (or your allowlisted set).

## Common pitfalls

1. **Skipping the advisory phase and going straight to enforce.** Almost guaranteed to fail the first PR after wiring. Advisory is the cheap-rollout pattern; respect it.
2. **Forgetting the `Fail when deterministic-deps failed` final step.** Without it, the upstream `continue-on-error: true` on the scan step (used so SARIF still uploads on failure) lets the job pass silently.
3. **Setting `severity-threshold: high` to "make it quieter."** The threshold only suppresses _lower_ severity findings; it doesn't surface fewer high-severity ones. Set it to `low` (the strictest) and use the per-rule allowlist when a specific finding is legitimately a non-issue.
4. **Allowlisting whole files instead of specific rules.** An entry without a `ruleId` suppresses everything in that file — including new rules added in future versions. Always pin the allowlist to a specific `ruleId` (or `ruleId` + `line`) so future rule additions are still surfaced.
5. **Not running it locally.** `npx -y Ozark-Security-Labs/deterministic-deps@v1` is fast. Run it before pushing; catch findings before they hit CI.

## Real example

[`Ozark-Security-Labs/forkguard/.github/workflows/dependency-determinism.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.github/workflows/dependency-determinism.yml) wires the action in enforce mode at `severity-threshold: low` with `remote-validation: true`. Its [`.deterministic-deps.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.deterministic-deps.yml) is the minimal config — empty allowlist, two excluded report paths. The repo passes cleanly today; check its CI history for the advisory→enforce transition.
