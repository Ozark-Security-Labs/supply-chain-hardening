# Rule 2.2 — Hash-pin every Python requirement

> Tier 2 · Hardened

## Rule

For Python projects, every dependency entry that CI installs must be pinned to an exact version *and* include a SHA-256 hash. `pip` (and `uv pip`) refuse the install if any package's content doesn't match the hash.

## Why it matters

Tier 1 asked you to commit a lockfile. For Python that mostly means `poetry.lock`, `uv.lock`, or `Pipfile.lock` — and those tools track hashes by default. The gap Rule 2.2 closes is the very common case of a project using plain `pip` with a `requirements.txt` file: even with `==` pins, an attacker who replaces the artifact on PyPI (account takeover, registry mirror compromise, name-similar typosquat resolved by your private index) defeats the version pin because nothing checks what was actually downloaded.

`--hash=sha256:<digest>` makes the install content-addressable. `pip install --require-hashes -r requirements.txt` then fails if any downloaded tarball's hash doesn't match. The defended-against attack class is registry tampering between lock and install time — the same surface Sigstore's npm/Maven/PyPI signing efforts are aimed at, with hashes as the immediate mechanism that works today without any registry-side cooperation.

## How to do it

### With `uv` (recommended for new projects)

[`uv`](https://docs.astral.sh/uv/) compiles a hashed `requirements.txt` from a `pyproject.toml` (or a starter `requirements.in`):

```sh
uv pip compile pyproject.toml --generate-hashes -o requirements.txt
```

Each line in the output looks like:

```
flask==3.1.0 \
    --hash=sha256:d667207822eb83f1c4b50949b1623c8fc8d51f2341d65f72e1a1815397551136 \
    --hash=sha256:5f873c5184c897c8d9d1b05df1e3d01b14910ce69607a117bd3277098a5836ac
```

Two hashes per package — one for the wheel, one for the source distribution — because either may be selected at install time depending on the platform.

CI install:

```sh
uv pip install --require-hashes -r requirements.txt
# or, with the older toolchain
pip install --require-hashes -r requirements.txt
```

`--require-hashes` is the gate. Without it, hashes in the file are advisory; with it, missing or mismatched hashes fail the install.

### With `pip-tools`

[`pip-tools`](https://github.com/jazzband/pip-tools) does the same thing without `uv`:

```sh
pip-compile --generate-hashes --output-file requirements.txt requirements.in
```

Then in CI:

```sh
pip install --require-hashes -r requirements.txt
```

### With Poetry

Poetry's lockfile already tracks hashes; the equivalent install gate is:

```sh
poetry install --no-update --sync
```

`--no-update` refuses to modify `poetry.lock`; `--sync` removes packages not in the lockfile. Hash verification happens by default.

### With Pipenv

`pipenv sync` honors `Pipfile.lock` hashes. The `--system` flag is appropriate in containers and CI.

### Multiple requirements files

If you split runtime and dev dependencies (`requirements.txt`, `requirements-dev.txt`), compile each with `--generate-hashes` and install each with `--require-hashes` separately. The hashes from one file don't carry over.

## How to verify

- **`deterministic-deps` rules** (see the [rule catalogue](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md)):
  - `python/hash-pinned-requirement` (medium) — `requirements*.txt` lines without `--hash=` are flagged
  - `python/lockfile-required` (high) — `pyproject.toml` or `Pipfile` without `poetry.lock` / `uv.lock` / `Pipfile.lock`
- **Alternatives:** `pip-audit --require-hashes` checks the runtime state but doesn't enforce hash presence at commit time. Tooling like `safety` and `pyup` cover vulnerability scanning, not hash-pinning specifically.
- **Manual:**

  ```sh
  grep -vE '^\s*(#|--|$)' requirements.txt | grep -v -- '--hash=sha256:' \
    && echo "unhashed requirements found"
  ```

## Common pitfalls

1. **Generating hashes once, then editing `requirements.txt` by hand.** Hashes go stale silently; the install will fail when CI hits the mismatch, and the diff is unreadable. Always regenerate via `uv pip compile` / `pip-compile`; never edit a hashed `requirements.txt` directly.
2. **Forgetting `--require-hashes` in CI.** Without the flag, hashes are documentation, not enforcement.
3. **Mixing hashed and unhashed entries in the same file.** `pip` refuses to install with `--require-hashes` if any line is missing `--hash=`. Compile-and-commit the full file as a unit.
4. **Pinning `pip-tools` or `uv` itself loosely.** The tool that produces hashes is also a supply-chain risk. Pin it in a constraints file or use a pre-built image with the version pinned.
5. **Assuming `--hash=` works on git or local file references.** It doesn't — hash-pinning is for registry installs only. Git deps go via the `python/git-sha` rule (full commit SHA in the requirement spec); local file deps are versioned by the project's own commit.

## Real example

No public Ozark-Security-Labs project ships a hash-pinned Python `requirements.txt` today; the org's Python footprint is limited (`forkguard-rulegen` is the only Python project and ships via `pyproject.toml` without a registry-publish flow). For a strong external example, see [pip-tools' usage docs on hashes](https://pip-tools.readthedocs.io/en/stable/#using-hashes), which document `pip-compile --generate-hashes` as the canonical pattern. This is a known gap in the OSL example set; bringing `forkguard-rulegen` (or any future OSL Python project) to a hash-pinned baseline is a follow-up.
