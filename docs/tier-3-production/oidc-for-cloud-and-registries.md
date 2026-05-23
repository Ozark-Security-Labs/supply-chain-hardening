# Rule 3.4 — Use OIDC for cloud and registry pushes (no long-lived secrets)

> Tier 3 · Production

## Rule

Every release-time push to a cloud provider (AWS, GCP, Azure), a package registry (npm, PyPI, Maven Central, Cargo via crates.io trusted publishers), or anything else that authenticates the workflow uses GitHub Actions OIDC federation. No long-lived API keys, no service account JSON, no PyPI tokens are stored in repository secrets.

## Why it matters

A long-lived secret in `Settings → Secrets and variables → Actions` is the highest-value target in your supply chain. If it leaks — through a compromised step (Rule 1.5's discussion of `tj-actions/changed-files`), through an over-broad fork PR, through a contributor accidentally pasting it into Slack — it's valid until rotated. Rotation is a manual process that depends on people noticing, knowing where the key is used, and coordinating downtime.

OIDC federation makes the workflow's *identity* the authenticator instead. The workflow exchanges a short-lived (15-minute) GitHub-issued OIDC token for a short-lived cloud / registry credential, scoped to exactly what the workflow needs and expiring on its own. There is no static secret to leak; the trust is on the cloud side — "I trust the GitHub workflow at `org/repo` running from tag `v*` on the default branch to assume this role."

For supply-chain integrity, this collapses the rotation problem into an attestation problem: the cloud's IAM trust policy now declares which workflows are allowed to push, and the workflow identity (the same identity used by SLSA and sigstore signing in Rules 3.1 and 3.3) carries that authority for exactly the duration of the run.

## How to do it

The pattern is the same across providers: a trust policy on the cloud side names the GitHub workflow identity that's allowed to assume the role; the workflow uses an `aws-actions/configure-aws-credentials` (or equivalent) action to exchange the OIDC token for a short-lived credential; the rest of the job uses that credential.

### AWS

Set up an IAM OIDC identity provider for `token.actions.githubusercontent.com`. Create a role whose trust policy allows assume by your repo's workflow:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/tags/v*"
      }
    }
  }]
}
```

The `StringLike` on `sub` is the gate: only workflows running from `v*` tags can assume this role. Tighten further (specific workflow file, specific environment) as needed.

In the workflow:

```yaml
permissions:
  id-token: write   # OIDC token
  contents: read

jobs:
  push:
    runs-on: ubuntu-24.04
    steps:
      - uses: aws-actions/configure-aws-credentials@<sha>
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/your-release-role
          aws-region: us-east-1
      - run: aws s3 cp dist/release.tar.gz s3://your-bucket/releases/
```

### GCP

Use Workload Identity Federation. Create a workload identity pool and provider mapping the GitHub OIDC issuer, then bind a service account to it:

```yaml
- uses: google-github-actions/auth@<sha>
  with:
    workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/github/providers/github'
    service_account: 'releaser@my-project.iam.gserviceaccount.com'
```

### Azure

Federated credentials on an app registration:

```yaml
- uses: azure/login@<sha>
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

Note `vars`, not `secrets` — the IDs aren't sensitive; the federation makes them sufficient.

### npm Trusted Publishers

Since 2024, [npm supports OIDC-based publishing via Trusted Publishers](https://docs.npmjs.com/trusted-publishers). Once configured at npmjs.com, you `npm publish --provenance` from the workflow with `id-token: write` permissions and no `NPM_TOKEN` — the provenance attestation includes the trust binding. See Rule 3.5 for the publishing-side details.

### PyPI Trusted Publishers

PyPI has the [same mechanism via PEP 740](https://docs.pypi.org/trusted-publishers/). [`pypa/gh-action-pypi-publish`](https://github.com/pypa/gh-action-pypi-publish) handles the OIDC exchange transparently.

### crates.io trusted publishing

Rust's [crates.io trusted publishing](https://github.com/rust-lang/rfcs/pull/3691) (RFC-led, in rollout as of mid-2025) follows the same pattern; the `actions-rs/cargo` ecosystem will expose it once stable.

## How to verify

- **`deterministic-deps`** doesn't enforce this — secrets vs OIDC is a release-flow choice, not a static configuration finding.
- **Alternatives:** the cloud-provider-specific OIDC actions ([`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials), [`google-github-actions/auth`](https://github.com/google-github-actions/auth), [`azure/login`](https://github.com/azure/login)) are the canonical paths. Third-party action wrappers exist but add a layer of trust.
- **Manual:** audit your repository secrets. Anything that says `*_TOKEN`, `*_KEY`, `*_SECRET` for a cloud or registry is a candidate for OIDC migration. The pattern is to migrate one at a time and delete the secret afterward — confirming via a release run that the OIDC path works before removing the fallback.

## Common pitfalls

1. **`audience` mismatch.** AWS requires `aud: sts.amazonaws.com`; GCP requires `aud: <workload-identity-pool-uri>`; npm Trusted Publishers requires `aud: npm:registry.npmjs.org`. Setting the wrong one produces an opaque 403; check the action's docs for the exact expected `audience`.
2. **Trust policy that's too broad.** A policy that allows `repo:your-org/your-repo:*` lets ANY workflow in the repo assume the role — including PR-time workflows that should not. Restrict by `ref` (`refs/tags/v*`), by environment (`environment:production`), or by workflow path (`job_workflow_ref:` claim).
3. **Forgetting `id-token: write` on the job.** Without it, the OIDC token endpoint returns nothing and the federation action fails with "no token available."
4. **Mixing OIDC and long-lived secrets in the same flow.** A release workflow that uses OIDC for the push but `NPM_TOKEN` for the publish leaves the high-value secret in place; an attacker who compromises the publish step has the token. Migrate the entire critical path.
5. **PR-time workflows running against OIDC-protected roles.** PR workflows for forks don't have OIDC access (the token is downgraded). Either gate OIDC-using jobs on `if: github.event_name != 'pull_request'`, or design the trust policy so PR workflows can't satisfy it.

## Real example

No public Ozark-Security-Labs project currently federates to a cloud provider via OIDC — the org's release pipelines today are entirely GitHub-internal (Releases, Marketplace, GitHub Packages). For a strong external example, see [`aws-actions/configure-aws-credentials`'s README](https://github.com/aws-actions/configure-aws-credentials#configuring-iam-to-trust-github), which walks through the AWS-side trust policy and the workflow-side configuration end to end. Adding OIDC to a future OSL release flow that pushes outside GitHub is the natural next step; the `id-token: write` permissions across SessionScope, AuthMap, and `deterministic-deps`' security workflows already exist for SLSA + sigstore, so the cost is the trust policy on the cloud side.
