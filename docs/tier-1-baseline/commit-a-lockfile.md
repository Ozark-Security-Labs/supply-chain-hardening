# Rule 1.1 — Commit a lockfile

> Tier 1 · Baseline

## Rule

For every package manager your project uses, commit the lockfile to the repository and install from it in CI in a mode that fails if the lockfile would be modified.

## Why it matters

Without a committed lockfile (or with one that CI silently regenerates), every install resolves version ranges against the registry's current state. If a maintainer of a transitive dependency publishes a malicious version one minute before your build runs, your build picks it up. With a committed lockfile and a frozen install, every CI run reproduces the exact dependency tree you reviewed at lock time. Nothing new lands until you intentionally update.

The canonical incident is the 2018 [`event-stream` compromise](https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident): a transitive package was handed off to a new "maintainer" who pushed a malicious update that targeted bitcoin wallets in `copay`. Any project without a lockfile pinned at `< 3.3.6` picked the malicious version up on its next install. Projects with a lockfile didn't — until they ran an unscoped update.

## How to do it

Per ecosystem, the lockfile to commit and the install command that refuses to modify it:

| Ecosystem | Lockfile to commit | CI install (frozen mode) |
| --- | --- | --- |
| npm | `package-lock.json` (use `lockfileVersion: 3`) | `npm ci` |
| pnpm | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| Yarn (Classic) | `yarn.lock` | `yarn install --frozen-lockfile` |
| Yarn (Berry) | `yarn.lock` + `.yarnrc.yml` | `yarn install --immutable` |
| pip + pip-tools | `requirements.txt` with `--hash=` lines | `pip install --require-hashes -r requirements.txt` |
| pip + uv | `uv.lock` | `uv sync --frozen` |
| Poetry | `poetry.lock` | `poetry install --no-update` |
| Pipenv | `Pipfile.lock` | `pipenv sync` |
| Cargo | `Cargo.lock` | `cargo build --locked` / `cargo test --locked` |
| Go | `go.sum` | `go build ./...` (verification is automatic when `go.sum` is present) |
| Bundler | `Gemfile.lock` | `bundle install --frozen` |
| Gradle | `gradle.lockfile` _and/or_ `gradle/verification-metadata.xml` | `./gradlew build` after locks are committed |

Three ecosystems need extra setup beyond "use the package manager normally":

**Python without a real lockfile.** A `requirements.txt` containing `flask>=2.0` is not a lockfile — transitive versions can drift, and there's nothing tying installed bytes to reviewed ones. Either move to [`pip-tools`](https://github.com/jazzband/pip-tools) (`pip-compile --generate-hashes`) or [`uv`](https://docs.astral.sh/uv/) (`uv pip compile --generate-hashes`) so every line includes `==exact.version` _and_ `--hash=sha256:...`. For application projects, `poetry` and `uv` manage `poetry.lock` / `uv.lock` natively without that extra step.

**Gradle.** Dependency locking is opt-in. Enable it once in `build.gradle.kts`:

```kotlin
dependencyLocking {
  lockAllConfigurations()
}
```

Then run `./gradlew dependencies --write-locks` to generate `gradle.lockfile` per project and commit those files. For stronger guarantees (checksum and signature verification of resolved artifacts), also commit `gradle/verification-metadata.xml` produced by `./gradlew --write-verification-metadata sha256`.

**Maven** has no native lockfile concept. You can pin versions exactly in `<dependencyManagement>`, but transitive resolution still happens at install time. For determinism-grade Maven builds, combine `<dependencyManagement>` pins on every transitive coordinate with the [Maven Enforcer plugin's `dependencyConvergence`](https://maven.apache.org/enforcer/enforcer-rules/dependencyConvergence.html) rule, which fails the build if any resolved version doesn't match the pin.

## How to verify

- **`deterministic-deps` rules** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `node/lockfile-required` (high) — `package.json` without a lockfile
  - `node/lockfile-coverage` (medium) — registry deps in the lockfile missing integrity hashes
  - `python/lockfile-required` (high) — `pyproject.toml` or `Pipfile` without a lockfile
  - `python/hash-pinned-requirement` (medium) — `requirements*.txt` entries without `--hash=`
  - `go/sum-required` (high) — `go.mod` without `go.sum`
  - `rust/lockfile-required` (high) — `Cargo.toml` without `Cargo.lock`
  - `ruby/lockfile-required` (high) — `Gemfile` without `Gemfile.lock`
  - `jvm/dynamic-version` (medium) — Maven or Gradle declarations using `SNAPSHOT`, `latest.*`, `+`, or a range without committed Gradle locking or verification metadata
- **Alternatives:** OpenSSF Scorecard's [`pinned-dependencies`](https://github.com/ossf/scorecard/blob/main/docs/checks.md#pinned-dependencies) check covers GitHub Actions and Dockerfile pins but does not enforce per-language lockfiles. `npm audit`, `pip-audit`, and `cargo audit` find vulnerable installs but won't flag a missing lockfile.
- **Manual one-liners** for the no-CI case:

  ```sh
  # npm: lockfile present and at lockfileVersion >= 3
  [ -f package-lock.json ] && jq -e '.lockfileVersion >= 3' package-lock.json >/dev/null

  # Python: every requirement line carries --hash=
  ! grep -vE '^\s*(#|--|$)' requirements.txt | grep -v -- '--hash=sha256:'

  # Go: go.mod implies go.sum
  ! { [ -f go.mod ] && [ ! -f go.sum ]; }

  # Rust: application/workspace crate has Cargo.lock
  ! { [ -f Cargo.toml ] && ! grep -q '^\[workspace\]\|^\[\[bin\]\]\|^name *=' Cargo.toml && [ ! -f Cargo.lock ]; }
  ```

## Common pitfalls

1. **`.gitignore`-ing the lockfile to reduce diff noise.** This is exactly backwards — the lockfile _is_ the security guarantee. Commit it. Use grouped [Dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#groups) or Renovate PRs if churn is the real problem.
2. **Treating `requirements.txt` as a lockfile.** It isn't — unless every line uses `==exact.version` _and_ `--hash=sha256:...`. Without hashes, a malicious tarball replacement on PyPI defeats your "pin" because the version string still matches.
3. **`npm install` in CI instead of `npm ci`.** `npm install` silently rewrites `package-lock.json` if `package.json` has changed; the build still passes and the divergence ships. `npm ci` refuses to run if the lockfile and `package.json` disagree.
4. **Cargo: dropping `Cargo.lock` from library crates.** [Cargo's recommendation since 2023](https://blog.rust-lang.org/2023/08/29/committing-lockfiles.html) is to commit `Cargo.lock` for every crate, including libraries. The library's own CI then runs against a known dependency tree, even though downstream consumers use their own lock.
5. **Gradle: assuming `dependencies { … }` pins are enough.** Without `dependencyLocking { lockAllConfigurations() }` and a committed `gradle.lockfile`, transitive resolution still happens fresh at every build, and a malicious update three levels deep lands without warning.

## Real example

[`Ozark-Security-Labs/deterministic-deps/package-lock.json`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/590b60e05138c3a56918f66078cb8759646c7cae/package-lock.json) — `lockfileVersion: 3` with per-entry integrity hashes. Its CI workflow at [`.github/workflows/ci.yml`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/590b60e05138c3a56918f66078cb8759646c7cae/.github/workflows/ci.yml) installs with `npm ci`, so any drift between `package.json` and `package-lock.json` fails the build immediately.
