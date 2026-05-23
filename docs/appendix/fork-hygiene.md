# Appendix B — Fork hygiene

When a transitive dependency starts looking risky — unresponsive maintainer, account-takeover red flags, install footprint that doesn't match the surface area you actually use — forking it is sometimes the right move. This appendix is the decision framework and the pointer to the existing Ozark Security Labs runbook for the procedure.

## When to fork (vs. pin, vs. replace)

Three reasonable responses to a worrying dependency:

| Response | When it fits |
| --- | --- |
| **Pin and wait** | The dep is generally healthy; the recent change is the only red flag; pinning to the last-known-good version buys time to assess |
| **Replace** | Another package covers your use case with better hygiene; switching is a one-shot effort that ends the relationship |
| **Fork-and-trim** | Replacement isn't viable (the API surface is too entangled), pinning indefinitely is fragile (the upstream may not get fixed), and the dep does just enough that you can vendor a stripped-down version under your own control |

Fork-and-trim costs more than pin-and-wait and less than replace. The recurring cost is keeping the fork's security patches current (which means watching the upstream for what's worth backporting); the one-time cost is the trim — removing functionality you don't use and the test surface around it.

## The Ozark Security Labs pattern

OSL prefixes its forks with `osl-`. The convention is enforced at three levels: repository name (`osl-glob`), package name (`@ozark-security-labs/osl-glob` or equivalent), and import-site references (so it's visible in code review that a non-upstream version is in use). This is the surface signal that a dependency was deliberately replaced, not accidentally divergent.

A complete list of current OSL forks lives in [`Ozark-Security-Labs/.github/blob/main/docs/ozark-stdlib.md`](https://github.com/Ozark-Security-Labs/.github/blob/main/docs/ozark-stdlib.md). Each entry names the upstream, the fork's SHA pin, and the rationale.

## The fork-and-trim workflow

The procedural runbook for proposing and shipping a fork lives in [`Ozark-Security-Labs/.github/blob/main/docs/fork-and-trim-workflow.md`](https://github.com/Ozark-Security-Labs/.github/blob/main/docs/fork-and-trim-workflow.md). Summary of the steps for readers without access:

1. **Write a fork proposal** (one-page document: upstream, why fork, what gets trimmed, who maintains)
2. **Fork into the org under `osl-<upstream-name>`** — repository visibility matches upstream's
3. **Trim aggressively** — remove every feature the consumer projects don't use; the goal is *less* code, not a friendly mirror
4. **Pin via git URL with full commit SHA** in the consumer projects (this is the Rule 1.1 lockfile angle applied to the fork itself)
5. **Periodically sync security fixes** from upstream; never sync features

The full doc covers the proposal template, the trim discipline, the CI requirements for OSL-published forks (Tiers 1 and 2 from this guide), and the upstream-tracking expectations.

## What forking does NOT solve

- **A genuinely malicious upstream.** Forking at a "last good" SHA before the compromise gives you a clean starting point, but you've also taken on permanent maintenance of code you didn't write. The same energy spent replacing the dependency with a different package is often better invested.
- **Disagreements about features.** Forking because you want a feature the upstream rejected is a maintenance trap. The fork drifts; sync becomes impossible; bug fixes don't flow back. Live with the disagreement or upstream the feature.
- **License problems.** A fork inherits the upstream license. If you're forking to escape AGPL, you can't; the fork is still AGPL.

## When the fork should be retired

Forks are work. Retire one when:

- Upstream regains active maintenance and ships a release that obsoletes your trim, OR
- A replacement library lands that covers the use case with better hygiene, OR
- The consumer projects no longer need the dependency at all

Don't keep a fork running because it's there. Each fork is its own Tier 1–3 supply-chain surface that has to be maintained alongside the consumer projects.
