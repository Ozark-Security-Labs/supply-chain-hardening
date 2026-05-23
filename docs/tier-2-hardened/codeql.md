# Rule 2.7 — CodeQL on push and PR

> Tier 2 · Hardened

## Rule

Every push and every PR runs [CodeQL](https://codeql.github.com/) against the project's source code in every supported language the repo uses. Findings flow into GitHub Code Scanning alongside `deterministic-deps` (Rule 2.1) and Scorecard (Tier 3) so all static results share one inbox.

## Why it matters

Tiers 1 and 2 up to this point are about the supply chain — what gets installed, what runs at build time, who can do what with which credentials. **CodeQL is the part that scans the project's own code.** It catches first-party security regressions (path traversals, injection sinks, unsafe deserialisation, certificate validation skips, hardcoded credentials) before they ship.

For a project hardening its supply chain, first-party static analysis is the complementary discipline: it doesn't matter how careful you are about which packages you depend on if your own code has the bug. Code Scanning is the unified surface that brings both classes of finding into one place; CodeQL is the GitHub-native engine for the first-party half.

CodeQL is free for public repos and for org plans that include GitHub Advanced Security. It's reasonable to run it on private repos that don't have GHAS — the action runs, the SARIF can be inspected as a workflow artifact, but the findings don't surface in the Code Scanning UI.

## How to do it

A workflow at `.github/workflows/codeql.yml`:

```yaml
name: CodeQL

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "31 10 * * 1"

permissions:
  contents: read
  security-events: write

jobs:
  analyze:
    name: Analyze (${{ matrix.language }})
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      matrix:
        language: [go]  # add 'javascript-typescript', 'python', 'java-kotlin',
                       # 'csharp', 'ruby', 'swift' as your repo uses them
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v5.0.0
        with:
          persist-credentials: false
      - uses: github/codeql-action/init@<sha>
        with:
          languages: ${{ matrix.language }}
          # Default queries are 'security-extended'; bump to 'security-and-quality'
          # if you can tolerate the extra findings:
          # queries: security-and-quality
      - name: Build  # only for compiled languages
        run: go build ./...
      - uses: github/codeql-action/analyze@<sha>
        with:
          category: "/language:${{ matrix.language }}"
```

Three choices matter:

**Language matrix.** CodeQL supports compiled (Go, Java/Kotlin, C/C++, C#, Swift) and interpreted (JavaScript/TypeScript, Python, Ruby) languages. The matrix entry is the [language identifier](https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning/configuring-advanced-setup-for-code-scanning#changing-the-languages-that-are-analyzed) — `javascript-typescript`, not `javascript`. Multi-language repos list each.

**Build step.** Compiled languages need the project to actually build during the CodeQL run; that's where the analyzer hooks in. Interpreted languages skip the build step. Most Go and Rust projects need a single `go build ./...` or `cargo build`; for monorepos or non-trivial build systems, use the [autobuild](https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning/configuring-advanced-setup-for-code-scanning#about-codeql-build-mode) helper as a starting point.

**Query suite.** `security-extended` (default) is broad enough to catch real issues without an overwhelming false-positive rate. `security-and-quality` adds maintainability findings — useful for a maturing codebase, noisy for a freshly hardening one. Start with the default; promote when the inbox is empty.

### Schedule

The cron line above runs CodeQL weekly even when no PR or push fires. CodeQL's rule database updates regularly; the weekly run catches new advisories that apply to your code without needing a code change.

### Triage workflow

CodeQL findings appear at **Repo → Security → Code scanning**. Each finding has a severity, a rule ID, a code location, and (for many rules) a remediation suggestion. The triage pattern that scales:

- **Blocking-severity findings** get fixed in a PR labelled `security`. The CodeQL alert auto-resolves when the offending code is removed.
- **Lower-severity findings** get triaged in a project board; the alert stays open until handled or explicitly dismissed.
- **False positives** are dismissed with a reason from the dropdown. CodeQL learns from dismissals across orgs.

Pair CodeQL with `deterministic-deps` (Rule 2.1) and Dependency Review (Rule 2.4): three checks, one Code Scanning inbox, with the entire dependency surface (third-party advisories, determinism, first-party code) covered.

## How to verify

- **`deterministic-deps`** is the determinism-static counterpart; not a substitute. Both belong on every PR.
- **Alternatives:**
  - [Semgrep](https://semgrep.dev/) — comparable static analyzer with a different rule ecosystem; especially strong for custom organization-wide rules. Free for public repos.
  - [SonarQube / SonarCloud](https://www.sonarsource.com/) — broader code-quality surface with security rules.
  - [Snyk Code](https://snyk.io/product/snyk-code/) — proprietary, integrated with Snyk's SCA.
  - For Go specifically, [`golangci-lint`](https://golangci-lint.run/) covers many security lints via plug-in linters; complementary rather than substitute.
- **Manual:** CodeQL CLI runs locally with [`codeql database analyze`](https://docs.github.com/en/code-security/codeql-cli/getting-started-with-the-codeql-cli) for diagnostics. Useful when triaging a specific finding.

## Common pitfalls

1. **Listing the wrong language identifier.** `javascript` is rejected; `javascript-typescript` is the current name. Check the [supported languages page](https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning/configuring-advanced-setup-for-code-scanning#changing-the-languages-that-are-analyzed) when adding a language to the matrix.
2. **Forgetting the build step for compiled languages.** CodeQL's analyzer needs to hook the build to extract the IR. Without a build step, Go and Rust analyses fail with a confusing "no source code found" error.
3. **Treating the weekly cron as redundant with PR runs.** It isn't. The rule database updates often; a clean PR run six months ago doesn't mean clean today against current rules.
4. **Dismissing findings without a reason.** The "Dismiss" dropdown distinguishes false positives, risk-accepted, and won't-fix. Pick the right one — it feeds the global signal CodeQL uses to refine rules.
5. **Running CodeQL but never opening the Code Scanning page.** The SARIF lands, the alerts pile up, and nobody acts on them. Pair the wire-in PR with a "triage on Mondays" calendar block until the inbox is at zero.

## Real example

[`Ozark-Security-Labs/forkguard/.github/workflows/codeql.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.github/workflows/codeql.yml) runs CodeQL on Go with a weekly cron, no matrix (single-language repo), and the default query suite. [`Ozark-Security-Labs/SessionScope/.github/workflows/codeql.yml`](https://github.com/Ozark-Security-Labs/SessionScope/blob/30df1c79544a9af6de6ab6eb0e39e7c5b219690e/.github/workflows/codeql.yml) is the equivalent for Rust.
