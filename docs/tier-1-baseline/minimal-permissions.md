# Rule 1.5 — Minimal workflow permissions

> Tier 1 · Baseline

## Rule

Every workflow file declares a top-level `permissions:` block that scopes `GITHUB_TOKEN` to read-only by default. Jobs that need more escalate explicitly per-job.

## Why it matters

`GITHUB_TOKEN` is injected into every workflow run by GitHub. Without an explicit `permissions:` block, the token's scope is governed by the repo's "Workflow permissions" setting — and many repositories (especially older ones) still default to **read and write**, which means every step in every workflow can push code, modify issues and PRs, edit deployments, and upload to packages. If any third-party action in your chain gets compromised (see Rule 1.2's discussion of the [tj-actions/changed-files](https://www.stepsecurity.io/blog/harden-runner-detection-tj-actions-changed-files-action-is-compromised) incident), the blast radius is whatever `GITHUB_TOKEN` could do.

An explicit `permissions: { contents: read }` at the top of every workflow caps the token's authority. The tj-actions payload still ran on compromised repos — but on the ones with minimal permissions, what it could exfiltrate or modify with that token was bounded. Defense in depth.

The default for new orgs is now read-only, but old orgs and unmigrated repos remain on read-and-write, and the repo-level setting can be overridden by the workflow. Declaring permissions explicitly in every workflow means the workflow's behaviour doesn't depend on whichever default happens to be set at the org or repo level today.

## How to do it

The minimum-viable block at the top of every workflow:

```yaml
permissions:
  contents: read
```

When a specific job needs more — uploading SARIF to Code Scanning, posting PR comments, pushing a release — declare it on the job, not at the top level. The job-level block **replaces** the workflow-level block (it is not additive), so list every scope the job needs:

```yaml
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-24.04
    # inherits permissions: contents: read

  scorecard:
    runs-on: ubuntu-24.04
    permissions:
      contents: read           # still needed; job-level replaces workflow-level
      security-events: write   # upload SARIF
      id-token: write          # OIDC for Scorecard publish_results
      actions: read
    steps:
      ...
```

Repo-level guardrail: in **Settings → Actions → General → Workflow permissions**, choose "Read repository contents and packages permissions" (the restrictive default). This caps what any workflow can ask for — a workflow asking for `contents: write` will still get `contents: write`, but a workflow with no `permissions:` block at all gets read-only instead of read-write. Set this at the org level too if you have one ("Organization → Settings → Actions → General").

The scopes available are documented at [GitHub Actions: Workflow permissions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#permissions-for-the-github_token). The common ones:

- `contents`: read repository content, push commits, manage releases
- `pull-requests`: read PRs, post review comments
- `issues`: read issues, post comments
- `security-events`: write SARIF to Code Scanning
- `id-token`: mint OIDC tokens (sigstore, cloud OIDC, SLSA generator, npm `--provenance`)
- `actions`: read workflow runs, cancel runs
- `packages`: read/write GitHub Packages

## How to verify

- **`deterministic-deps`** does not currently include a rule for workflow permissions; this is on the v1.x roadmap. Until then:
- **Alternatives:**
  - [OpenSSF Scorecard's `Token-Permissions` check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#token-permissions) (covered today). It fails when a workflow doesn't declare `permissions:` or when top-level permissions are over-broad.
  - [`step-security/secure-repo`](https://github.com/step-security/secure-repo) rewrites workflows to add minimal `permissions:` blocks alongside SHA pinning.
- **Manual one-liner:** flag any workflow file without a top-level `permissions:` declaration.

  ```sh
  for f in .github/workflows/*.yml .github/workflows/*.yaml; do
    [ -f "$f" ] || continue
    # Look for a permissions: key at column 0 (workflow-level) within the first 30 lines
    head -30 "$f" | grep -qE '^permissions:' || echo "$f: no top-level permissions"
  done
  ```

## Common pitfalls

1. **Setting top-level permissions, then a job-level block that drops a scope by accident.** Job-level `permissions:` replaces workflow-level; it does not extend. If your workflow has `permissions: { contents: read, id-token: write }` and a job declares `permissions: { contents: write }`, that job loses `id-token: write`. Always list every scope each job needs.
2. **Forgetting `id-token: write` for OIDC.** Sigstore signing, AWS / GCP OIDC, SLSA generators, and `npm publish --provenance` all need `id-token: write`. Without it, the call to the OIDC token endpoint fails with a confusing error.
3. **Relying on the org default and never declaring permissions in the workflow.** Org defaults change. Repos move between orgs. Whatever your default is today, declaring permissions in the workflow makes the behaviour portable.
4. **Trying to escalate permissions on PRs from forks.** GitHub forces `GITHUB_TOKEN` for fork PRs to read-only regardless of the workflow's `permissions:` declaration — this is a hardcoded platform behaviour, not configurable. Design fork-PR workflows around it (e.g., only the merged commit gets full permissions; the PR-trigger workflow does read-only validation).
5. **Confusing `permissions:` (token scopes) with `permissions:` on a `step` (runner sudo).** They are unrelated. Workflow/job `permissions:` controls `GITHUB_TOKEN` scope; there is no step-level `permissions:` key.

## Real example

[`Ozark-Security-Labs/forkguard/.github/workflows/dependency-determinism.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.github/workflows/dependency-determinism.yml) declares a workflow-level `permissions: { contents: read, security-events: write }` (the SARIF-upload pattern). The repo's [`ci.yml`](https://github.com/Ozark-Security-Labs/forkguard/blob/7ffbacb42cce5418e1e725b79114d305c469f0d0/.github/workflows/ci.yml) uses the more typical `permissions: { contents: read }` since it only checks out and tests.
