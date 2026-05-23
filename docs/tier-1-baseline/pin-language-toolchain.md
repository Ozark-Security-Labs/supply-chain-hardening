# Rule 1.4 — Pin the language toolchain

> Tier 1 · Baseline

## Rule

Pin every language toolchain your project uses (Node, Python, Go, Rust, Ruby, JVM) in a tracked file in the repo, and have your CI setup step read the version from that file.

## Why it matters

When `actions/setup-node` sees `node-version: '20'`, it installs whatever `20.x.y` the runner image considers current. That version moves silently as the runner image is refreshed and as Node's release cadence advances. The same is true for `setup-python`, `setup-go`, and every other `setup-*` action that accepts a version range.

For build correctness this is mostly an annoyance — tests start failing or, worse, start subtly producing different output. For supply chain it's a credibility hole at the Tier 3 layer: a [SLSA provenance attestation](https://slsa.dev/spec/v1.0/provenance) listing `node: 20.x` as the build tool tells a downstream verifier nothing about what compiled the artifact. Pinning the toolchain to a tracked file makes the "what compiled this" question answerable and the answer auditable through `git log`.

The version pin doesn't have to be patch-perfect to be useful. The thresholds, ranked best to worst:

| Granularity | Example | Notes |
| --- | --- | --- |
| **Exact patch** | `.nvmrc` containing `v22.11.0` | Strongest. Patch floats only when you intentionally update the file. |
| **Minor range** | `node-version: '22.11'` in workflow | Acceptable if combined with Dependabot/Renovate to keep the range moving. |
| **Major range** | `node-version: '22'` | Weakest defensible position. Patch and minor both float on each run. |
| **Floating** | `node-version: 'latest'`, `lts/*`, `stable` | Don't. |

Tier 1 asks for at least the minor range *and* a tracked file. Tier 3 effectively demands the exact patch.

## How to do it

The pattern is the same across ecosystems: commit a small file at the repo root that names the version, and tell the `setup-*` action to read from it. Per ecosystem:

| Ecosystem | File to commit | Setup-action reads via |
| --- | --- | --- |
| Node.js | `.nvmrc` containing e.g. `v22.11.0` | `actions/setup-node` with `node-version-file: '.nvmrc'` |
| Python | `.python-version` containing e.g. `3.13.1` | `actions/setup-python` with `python-version-file: '.python-version'` |
| Go | `go.mod` with `go 1.23.4` (Go 1.21+ accepts the full triple) | `actions/setup-go` with `go-version-file: 'go.mod'` |
| Rust | `rust-toolchain.toml` with `channel = "1.83.0"` | `rustup` (and `dtolnay/rust-toolchain`) read it automatically |
| Ruby | `.ruby-version` containing e.g. `3.3.6` | `ruby/setup-ruby` with `ruby-version-file: '.ruby-version'` |
| Java / JVM | `.tool-versions` (asdf/mise) or explicit `java-version: '21.0.5'` | `actions/setup-java` |
| Multi-language | `.tool-versions` for [`asdf`](https://asdf-vm.com/) / [`mise`](https://mise.jdx.dev/) | tool-specific setup actions, or `mise/mise-action` |

A concrete Node example:

```sh
echo 'v22.11.0' > .nvmrc
git add .nvmrc && git commit -m "Pin Node to 22.11.0"
```

```yaml
# .github/workflows/ci.yml
- uses: actions/setup-node@<sha>  # v6.0.0
  with:
    node-version-file: '.nvmrc'
    cache: npm
```

A concrete Go example using only `go.mod`:

```go
// go.mod
module github.com/your-org/your-repo

go 1.23.4
```

```yaml
- uses: actions/setup-go@<sha>  # v6.0.0
  with:
    go-version-file: 'go.mod'
```

A concrete Rust example:

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.83.0"
components = ["rustfmt", "clippy"]
```

```yaml
- uses: dtolnay/rust-toolchain@<sha>
  with:
    toolchain: '1.83.0'  # or omit and let rust-toolchain.toml win
```

Package-manager versions are a separate axis — pin those too:

- npm: `"packageManager": "npm@10.9.0"` in `package.json`. Corepack respects it; `npm ci` does not always.
- Yarn / pnpm: `"packageManager": "yarn@4.5.1"` / `"packageManager": "pnpm@9.12.3"`. Corepack honors this when enabled (`corepack enable` in CI).
- pip: `pip --version` is whatever the toolchain installer ships; pin `pip` itself via `pip install pip==24.3.1` if your tooling depends on a specific resolver.

## How to verify

- **`deterministic-deps` rule** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `rust/toolchain-version` (medium) — `rust-toolchain.toml` and legacy `rust-toolchain` files must not use floating `stable`, `beta`, or `nightly` channels. Dated channels (`nightly-2024-05-01`) and exact versions (`1.83.0`) are accepted.

  The other ecosystems aren't currently covered by a `deterministic-deps` rule — Node, Python, Go, Ruby, and JVM toolchain pinning is on the v1.x roadmap. Until then, the manual checks below carry the load.

- **Alternatives:** Renovate's `nodenv`, `python-version`, `rust-toolchain`, and `asdf` managers update these files in PRs; Dependabot does not maintain `.nvmrc` / `.python-version` style files today. OpenSSF Scorecard does not check toolchain pinning.
- **Manual one-liners:**

  ```sh
  # Node: .nvmrc present and exact (vMAJOR.MINOR.PATCH)
  [ -f .nvmrc ] && grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+$' .nvmrc

  # Python: .python-version present and exact
  [ -f .python-version ] && grep -qE '^[0-9]+\.[0-9]+\.[0-9]+$' .python-version

  # Go: go.mod has a full version triple (Go 1.21+)
  grep -qE '^go [0-9]+\.[0-9]+\.[0-9]+$' go.mod

  # Rust: rust-toolchain.toml channel is not 'stable'/'beta'/'nightly'
  grep -E '^\s*channel\s*=' rust-toolchain.toml | grep -vE '"(stable|beta|nightly)"'
  ```

## Common pitfalls

1. **Pinning a range in the workflow, not in a tracked file.** `node-version: '22'` in the YAML pins the workflow's intent but doesn't move with `git blame` the way a `.nvmrc` does, and developers running locally won't see the pin. Use `*-version-file` plus a tracked file as the source of truth.
2. **`.nvmrc` containing `lts/*` or `node`.** Both are floating; neither survives a node release. Use a specific `vX.Y.Z`.
3. **`rust-toolchain.toml` with `channel = "stable"`.** Same problem; flagged by `deterministic-deps` rule `rust/toolchain-version`. Use `1.83.0` or whatever exact version you want.
4. **Pinning the toolchain twice and letting them drift.** If your `go.mod` says `go 1.23.4` and your Dockerfile says `FROM golang:1.23.2`, you have two pins of the same thing and the build is whichever environment runs. Either single-source from `go.mod` (using `--build-arg GO_VERSION=$(awk '/^go / {print $2}' go.mod)`) or accept the duplication and add a CI check that they match.
5. **Forgetting the package manager.** A pinned Node version with an unpinned npm/yarn/pnpm leaves a moving piece in the install chain. Use the `packageManager` field in `package.json` (Corepack reads it) or pin via the setup action's input.

## Real example

[`Ozark-Security-Labs/PkgWarden/.github/workflows/ci.yml`](https://github.com/Ozark-Security-Labs/PkgWarden/blob/d4db739a5fb6c1a06fa034c6e9b365cf6e141135/.github/workflows/ci.yml) uses `actions/setup-go` with `go-version-file: go.mod`; the version is sourced from [`PkgWarden/go.mod`](https://github.com/Ozark-Security-Labs/PkgWarden/blob/d4db739a5fb6c1a06fa034c6e9b365cf6e141135/go.mod) (currently `go 1.23` — the version-triple form is a future tightening).
