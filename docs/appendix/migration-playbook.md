# Appendix C — Migration playbook

> Status: placeholder. Ships in **v1.0.0**.

Adopting these patterns on a brand-new project is straightforward. Retrofitting them onto a live codebase without burning out the contributor pool is the actual hard problem. This appendix lays out a tested order of operations:

1. Tier 1 first, all in one PR — it's mechanical and breaks nothing.
2. Wire `deterministic-deps` in advisory mode for at least one release cycle so the team sees what enforcement would block before it blocks.
3. Promote to enforce mode only after the noise level is zero or single-digit.
4. Add Tier 3 release machinery as a separate workstream, on a separate branch, behind a feature flag in the release workflow.
