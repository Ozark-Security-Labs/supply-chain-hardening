# Rule 1.3 — Pin the runner OS

> Tier 1 · Baseline

## Rule

In every workflow's `runs-on:` field, use a versioned runner label (`ubuntu-24.04`, `windows-2022`, `macos-14`), never the floating alias (`ubuntu-latest`, `windows-latest`, `macos-latest`).

## Why it matters

GitHub reassigns the `*-latest` labels periodically. When `ubuntu-latest` migrated from 22.04 to 24.04, every workflow using that alias swapped over within a deprecation window — and along with the OS upgrade came a fresh batch of pre-installed tool versions: `git`, `node`, `python`, `openssl`, `gcc`, `docker`, and several hundred more. Workflows that depended on a specific pre-installed version started failing or, worse, started succeeding while producing subtly different output.

For supply-chain integrity the consequence is sharper than build flakiness: a [SLSA provenance attestation](https://slsa.dev/spec/v1.0/provenance) that lists `ubuntu-latest` as the builder image is telling consumers nothing useful. They can't reproduce the build because they don't know what was on the runner. They can't trust the attestation because the runner identity is a moving target. Tier 3 hardening (provenance, reproducibility) is undermined at the root if Tier 1 isn't on a pinned runner.

This is the rare Tier 1 pattern where the immediate cost of getting it wrong is broken builds, not breached secrets — but it makes Tiers 2 and 3 work, so it belongs at the baseline.

## How to do it

Replace every floating alias with a versioned label:

```yaml
# Bad
jobs:
  test:
    runs-on: ubuntu-latest

# Good
jobs:
  test:
    runs-on: ubuntu-24.04
```

Current versioned labels GitHub publishes (check the [GitHub-hosted runners reference](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners/about-github-hosted-runners) for the live list):

| OS family | Pinned labels available | Floating alias to replace |
| --- | --- | --- |
| Linux | `ubuntu-24.04`, `ubuntu-22.04` | `ubuntu-latest` |
| Windows | `windows-2025`, `windows-2022` | `windows-latest` |
| macOS | `macos-15`, `macos-14`, `macos-13` | `macos-latest` |

In a matrix, enumerate explicit pinned labels rather than letting `${{ matrix.os }}` resolve to a floating alias:

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-24.04, macos-14, windows-2022]
runs-on: ${{ matrix.os }}
```

Two reference categories that don't need pinning:

- **Self-hosted and custom-labelled runners** (`self-hosted`, `[self-hosted, linux, x64]`, custom Larger Runner labels) — these are infrastructure you control, so the label is meaningful by construction.
- **Runner groups** — group references resolve at the org level, not against GitHub's hosted pool.

`deterministic-deps` recognises both categories and only flags GitHub's floating aliases.

### Renewal cadence

GitHub-hosted runner OS versions have published support windows. `ubuntu-22.04` and `macos-13` are old enough that their retirements are on the public roadmap. Add a calendar reminder to review your pinned labels yearly, or wire it into your Dependabot/Renovate flow (Renovate's [`github-actions` manager understands `runs-on:` labels](https://docs.renovatebot.com/modules/manager/github-actions/)). The goal is bounded, deliberate upgrades — not stasis.

## How to verify

- **`deterministic-deps` rule** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `github-actions/versioned-runner` (medium) — flags `ubuntu-latest`, `windows-latest`, and `macos-latest` in scalar `runs-on:`, array `runs-on:`, and matrix entries. Custom and self-hosted labels are explicitly allowed.
- **Alternatives:** the [`actions/runner-images` repo](https://github.com/actions/runner-images) tracks current and deprecated images (deprecation announcements land in its issue tracker), but there's no out-of-the-box GitHub-side check that fails a workflow using `*-latest`. OpenSSF Scorecard does not check runner labels today. `deterministic-deps` is the only popular static check.
- **Manual one-liner:**

  ```sh
  grep -rEn '^\s*runs-on:\s*([a-z]+-latest|\[[^]]*-latest)' .github/workflows/ \
    && echo "floating runner alias found"
  ```

## Common pitfalls

1. **Pinning the scalar `runs-on:` but leaving floating aliases in the matrix.** `deterministic-deps` checks both, but the manual one-liner above is array-aware and the human eye often misses matrix entries. Sweep both.
2. **Pinning the runner and assuming the toolchain comes along.** It doesn't — the pre-installed Node, Python, Go, etc. on `ubuntu-24.04` will move as the image is refreshed. Pin your language toolchain separately (Rule 1.4) and use the `actions/setup-*` action with an explicit version. The runner pin is the substrate; the toolchain pin sits on top.
3. **Treating runner version pins as permanent.** GitHub retires images. `ubuntu-22.04` will reach end-of-support; CI starts failing one day with no warning if you ignore the retirement schedule. Renew on a cadence, not on outage.

## Real example

[`Ozark-Security-Labs/deterministic-deps/.github/workflows/ci.yml`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/590b60e05138c3a56918f66078cb8759646c7cae/.github/workflows/ci.yml) — every job uses `ubuntu-24.04`, and the matrix entries (where present) enumerate pinned labels.
