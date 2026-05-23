# Rule 2.3 — Pin container image digests

> Tier 2 · Hardened

## Rule

Every reference to a container image — in `Dockerfile`, `compose.yml`, `devcontainer.json`, and any GitHub Action that pulls an image — uses `name:tag@sha256:<digest>`. Plain `name:tag` and bare `name` are forbidden; `latest` is forbidden.

## Why it matters

A container image tag is mutable in exactly the same way a git tag is. `python:3.13` today is a different content-addressable artifact than `python:3.13` next week — the registry advances the tag as new patch builds ship. For build reproducibility this means even a clean rebuild of an old `git checkout` produces different binaries; for supply-chain integrity it means a [registry compromise](https://www.docker.com/blog/docker-engine-security-update/) or a typosquat (`pyhton:3.13` resolving to an attacker-controlled name) goes undetected.

Adding `@sha256:<digest>` makes the reference content-addressable. The image's manifest hash is the identity; any tampering — at the registry, on the CDN, in flight — produces a different digest and the pull fails. This is exactly the SHA-pinning model from Rule 1.2, applied to containers. SLSA Level 3 effectively requires it for the build base image.

## How to do it

In a `Dockerfile`:

```Dockerfile
# Bad
FROM python:3.13

# Good
FROM python:3.13.1-slim@sha256:f3614d98f38b0f8d8b07d8d6f6e0fc6e1c34a8b94b8b6b76e6d97c0db1c5e3a9
```

You can keep the tag on the line alongside the digest. The registry serves the image identified by the digest; the tag is there for human readability. (Some scanners — including `deterministic-deps`' `containers/image-digest` rule — accept either `name@sha256:...` or `name:tag@sha256:...`.)

In `docker-compose.yml`:

```yaml
services:
  api:
    image: python:3.13.1-slim@sha256:f3614d98f38b0f8d8b07d8d6f6e0fc6e1c34a8b94b8b6b76e6d97c0db1c5e3a9
```

In `devcontainer.json`:

```json
{
  "image": "mcr.microsoft.com/devcontainers/python:1.2.3-3.13@sha256:..."
}
```

In a GitHub Action's `uses: docker://`:

```yaml
- uses: docker://ghcr.io/some/action@sha256:...
```

### Finding the digest for a tag

```sh
# Public registries (Docker Hub, ghcr.io, quay.io, mcr.microsoft.com):
docker buildx imagetools inspect python:3.13.1-slim | head -5

# Or with crane (smaller, no Docker daemon needed):
crane digest python:3.13.1-slim
# -> sha256:f3614d98f38b0f8d8b07d8d6f6e0fc6e1c34a8b94b8b6b76e6d97c0db1c5e3a9
```

For multi-platform images (`linux/amd64`, `linux/arm64`), `crane digest` returns the digest of the **manifest list** — the right thing to pin. Docker honors the list and selects the platform-specific image at pull time, all under one digest.

### Keeping digests current

[Renovate](https://docs.renovatebot.com/configuration-options/#pindigests) opens PRs that bump the digest when the underlying tag advances; Dependabot supports container digest updates for `Dockerfile` and `docker-compose.yml`. Pick one, enable it, and let the digests roll over on a cadence. The Tier 1 principle — bounded updates, not stasis — applies.

## How to verify

- **`deterministic-deps` rule** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `containers/image-digest` (medium / high) — Dockerfile, Compose, and devcontainer image references must use `name:tag@sha256:<digest>`. `latest` and untagged references are high severity; tagged-but-undigested references are medium.
- **Alternatives:** [Trivy](https://github.com/aquasecurity/trivy) flags un-pinned images alongside its vulnerability scan; [OpenSSF Scorecard's `pinned-dependencies` check](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) covers Dockerfile image pinning as part of its broader pin assessment.
- **Manual:**

  ```sh
  # Any FROM line without @sha256:
  grep -nE '^FROM\s+[^[:space:]]+(\s|$)' Dockerfile | grep -v '@sha256:'
  ```

## Common pitfalls

1. **Pinning the digest but forgetting the tag.** `FROM python@sha256:...` works but loses the human-readable version label. Keep both: `FROM python:3.13.1-slim@sha256:...`. The digest is the security mechanism; the tag is what you scan-read later.
2. **Pinning a platform-specific digest instead of the manifest list.** If you pin the `linux/amd64` digest, the same Dockerfile fails on `linux/arm64`. Use the manifest list digest (what `crane digest <tag>` returns by default) so multi-arch builds still resolve.
3. **Forgetting `docker-compose.yml` and devcontainers.** The `containers/image-digest` rule scans all three; CI in production scans Dockerfile only is a common oversight.
4. **Trying to pin private registry images without an automated bump path.** Manual digest updates rot. Either give Renovate or Dependabot the registry credentials, or run a scheduled job that bumps digests via PR.
5. **Pinning images that change beneath you in the registry.** Most public images are immutable per-digest, but some private registries allow digest deletion. Verify your registry of choice marks digests as immutable.

## Real example

No public Ozark-Security-Labs project currently ships a Dockerfile (the org's portfolio is CLI binaries, libraries, and GitHub Actions — none of which need a runtime container today). For a strong external example, see [Distroless'](https://github.com/GoogleContainerTools/distroless/blob/main/examples/python3/Dockerfile) reference Dockerfiles or the [`actions/runner-images`](https://github.com/actions/runner-images) generated Dockerfiles.

When an OSL project that publishes a container image lands (in scope for late 2026 per the org roadmap), this section will get a first-party real example.
