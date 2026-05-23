# Rule 2.6 — `persist-credentials: false` and harden-runner

> Tier 2 · Hardened

## Rule

Set `persist-credentials: false` on every `actions/checkout` step that doesn't need to push or call the GitHub API afterwards. On any workflow that runs builds or invokes third-party tooling, run [`step-security/harden-runner`](https://github.com/step-security/harden-runner) at the start of the job to monitor (and optionally block) egress traffic from compromised steps.

## Why it matters

These are two related defenses against the same attack class: **a compromised step exfiltrating something it shouldn't**. Rule 1.5 capped what `GITHUB_TOKEN` could do; Rule 2.6 caps what a step can do with everything else on the runner.

`actions/checkout` by default leaves `GITHUB_TOKEN` in the local git config so subsequent steps can `git push`. Most jobs don't push — they checkout, test, and exit. Leaving the token in `.git/config` means every later step in the job can read it (e.g., `cat .git/config`). The 2025 `tj-actions/changed-files` payload (Rule 1.2's reference incident) exfiltrated exactly this kind of data. `persist-credentials: false` strips the token from the local config immediately after checkout; subsequent steps that try to read it find an empty value.

`harden-runner` is the egress side: it installs a network filter on the runner that logs every outbound connection a step makes, optionally blocks connections to hosts not on an allowlist, and produces a per-run report visible in the GitHub Action summary. The malicious tj-actions payload tried to upload to a specific host; a harden-runner allowlist would have blocked the upload and surfaced the attempt in the action summary even though the malicious code had already run.

Both are defense-in-depth: they don't prevent compromise, they bound its consequences.

## How to do it

### `persist-credentials: false`

Set it on every `actions/checkout` step in every workflow:

```yaml
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v5.0.0
  with:
    persist-credentials: false
```

Two cases where you need credentials to persist:

1. The job pushes commits, tags, or releases. In that case, opt into the persistence on _that_ job only, and gate it with a clear comment:

   ```yaml
   - uses: actions/checkout@<sha>
     with:
       # Needed for release-please / git push back to main.
       persist-credentials: true
       fetch-depth: 0
   ```

2. The job calls `gh` or the GitHub API and expects `git`-style credentials. Modern `gh` reads `GH_TOKEN` from the environment, not from `.git/config`, so this is almost never necessary. Prefer passing `GH_TOKEN: ${{ github.token }}` explicitly.

Make `persist-credentials: false` the default; opt into persistence per job; comment the opt-in.

### `step-security/harden-runner`

Add it as the first step of any job that runs untrusted tooling. Start in audit mode to see what egress your build actually makes:

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04
    steps:
      - uses: step-security/harden-runner@<sha>  # see action README for current SHA
        with:
          egress-policy: audit
          disable-sudo: true
          disable-file-monitoring: false

      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd
        with:
          persist-credentials: false

      - run: npm ci
      - run: npm test
```

After one or two release cycles in audit mode, inspect the per-run "Insights → Harden-Runner" summary to see the set of egress endpoints your build actually contacts (registry hosts, telemetry endpoints, download mirrors). Promote to block mode with that set as the allowlist:

```yaml
      - uses: step-security/harden-runner@<sha>
        with:
          egress-policy: block
          allowed-endpoints: |
            api.github.com:443
            codeload.github.com:443
            objects.githubusercontent.com:443
            registry.npmjs.org:443
            pypi.org:443
            files.pythonhosted.org:443
```

The allowlist is a per-host:port set, not URLs. Anything outside fails the connection (and is logged with the attempted host so you can identify a legitimate omission vs. a malicious attempt).

### Combine with `disable-sudo`

`disable-sudo: true` (default `false` for backwards compat) removes the runner user's `sudo` access for the rest of the job. Most build steps don't need sudo. Removing it means a step that tries `sudo apt install` fails — and a step that tries to escalate privileges as part of an exploit chain fails.

## How to verify

- **`deterministic-deps`** does not cover these two patterns today. They're on the v1.x roadmap as separate rules.
- **Alternatives:**
  - [OpenSSF Scorecard's `Token-Permissions` check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#token-permissions) flags missing `permissions:` blocks (Rule 1.5) but does _not_ flag missing `persist-credentials: false`.
  - Scorecard's [`Pinned-Dependencies`](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) check confirms harden-runner is SHA-pinned but doesn't require its presence.
  - Step Security's own [`secure-repo`](https://github.com/step-security/secure-repo) browser tool will rewrite workflows to add both patterns.
- **Manual:**

  ```sh
  # Find any actions/checkout step without persist-credentials: false
  grep -rB 1 'actions/checkout@' .github/workflows/ | grep -A 3 'uses:' \
    | grep -v 'persist-credentials: false' | grep 'uses: actions/checkout'
  ```

## Common pitfalls

1. **`persist-credentials: false` on a job that pushes.** The push silently fails or, worse, the job times out waiting for git auth. Audit per job; only the jobs that push should keep the credential.
2. **Going straight to `egress-policy: block`.** Almost guaranteed to fail your first build because some legitimate endpoint isn't on the allowlist. Use `audit` mode for two release cycles before flipping to `block`.
3. **Treating the harden-runner allowlist as static.** Registry mirrors get added, CDN hostnames rotate. Plan to refresh the allowlist on a quarterly cadence (or on a CI failure that points at a legitimate-looking endpoint).
4. **Forgetting that harden-runner is itself a third-party action.** SHA-pin it like any other (Rule 1.2) and add it to your Dependabot/Renovate config (Rule 2.5) so the SHA bumps with the rest.
5. **Mixing block-mode harden-runner with Dependabot's network calls.** Dependabot doesn't run on the same runner as your workflows, but its security-update PRs trigger your workflows on push — and those workflows then go through harden-runner. Make sure your allowlist covers the package registries Dependabot pulls from for the diff scan.

## Real example

`persist-credentials: false`: [`Ozark-Security-Labs/PkgWarden/.github/workflows/scorecard.yml`](https://github.com/Ozark-Security-Labs/PkgWarden/blob/d4db739a5fb6c1a06fa034c6e9b365cf6e141135/.github/workflows/scorecard.yml) sets it on the checkout step, which is the typical Scorecard pattern.

`harden-runner`: no public OSL repo uses it today. The recommended Tier 2 worked example is [`step-security/harden-runner`](https://github.com/step-security/harden-runner)'s own README — it's the canonical reference. Adding harden-runner to one of the existing OSL repos (likely `deterministic-deps`, since it's our own tooling) is a near-term follow-up.
