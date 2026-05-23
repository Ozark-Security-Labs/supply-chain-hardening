# Tier 1 — Baseline

> Status: outline only. Patterns ship in **v0.1.0** (see [project status](../../README.md#status)).

Tier 1 is the floor every repo should clear on day one. It costs almost nothing and delivers the largest single jump in supply-chain integrity: known dependencies, known build inputs, and CI tokens scoped to what each job actually needs.

Worked example throughout this tier: [`Ozark-Security-Labs/osl-glob`](https://github.com/Ozark-Security-Labs/osl-glob).

## Planned patterns

1. **Commit a lockfile** — npm/pnpm/yarn, pip/poetry/uv, cargo, go, bundler, gradle
2. **SHA-pin every third-party GitHub Action** — full 40-character commit SHAs, never tags or branches
3. **Pin the runner OS** — `ubuntu-24.04`, not `ubuntu-latest`
4. **Pin the language toolchain** — `.nvmrc`, `.python-version`, `rust-toolchain.toml`
5. **Minimal workflow permissions** — `permissions: { contents: read }` as the default; widen per job only when needed

Each pattern will follow the [per-pattern entry template](../../CONTRIBUTING.md#per-pattern-entry-template) once written.

## Next tier

When the patterns above are routine for you, move on to [Tier 2 — Hardened](../tier-2-hardened/).
