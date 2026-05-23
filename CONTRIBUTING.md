# Contributing to Supply Chain Hardening

Thank you for considering contributing. This guide lives on GitHub at <https://github.com/Ozark-Security-Labs/supply-chain-hardening> and is mirrored to <https://docs.ozarksecuritylabs.com/supply-chain/> via the `docs-hub` Astro Starlight site. **The GitHub repo is the canonical source** — edits land here first, the rendered site follows.

## What kinds of contributions are welcome

| Type | How to start |
| --- | --- |
| New pattern in an existing tier | Open an issue with the pattern title and the attack class it addresses; if the maintainers agree it fits, send a PR using the template below |
| Correction or clarification | Send a PR directly — keep the change tight |
| Alternative tool for an existing pattern | Send a PR against `docs/appendix/tool-landscape.md`. Be honest about tradeoffs — we list alternatives by design |
| A worked example from your own repo | Open an issue first; we curate the "Real example" links to a maintained set so the doc doesn't rot when a referenced repo goes stale |
| Translation | Open an issue first to coordinate file layout |

## Per-pattern entry template

Every pattern in Tiers 1–3 uses this exact structure. Copy it verbatim when proposing a new pattern; reviewers check that all sections are present and non-empty.

````markdown
## Rule N.N — <one-line imperative>

### Rule
<One sentence. Action-oriented. No qualifiers.>

### Why it matters
<The attack class this prevents. One or two sentences. Link to a
representative CVE or public incident where one exists. No
threat-modeling exposition.>

### How to do it
<Copy-pasteable config or YAML. Use ecosystem tabs where the recipe
differs (npm/pnpm/yarn, pip/poetry/uv). Snippets live verbatim in
/examples/<category>/<filename> and are referenced here with a relative
link rather than inlined twice.>

### How to verify
- **`deterministic-deps` rule:** [`<rule-id>`](https://github.com/Ozark-Security-Labs/deterministic-deps/blob/main/docs/rules.md#<anchor>) (<severity>)
- **Alternatives:** <tool>, <tool>
- **Manual:** <one-line grep, lint, or jq the reader can run with no CI>

### Common pitfalls
1. <gotcha>
2. <gotcha>

### Real example
[`<repo>/<path>`](https://github.com/Ozark-Security-Labs/<repo>/blob/<commit-sha>/<path>)
````

## Style

- **Tense:** imperative present ("pin every action," not "you should pin"). Direct, no padding.
- **Voice:** maintainer to maintainer. Assume the reader has shipped code, just not this kind of hardening.
- **Snippets:** complete and copy-runnable. Avoid `…` or `<your-thing>` placeholders unless the placeholder is genuinely user-specific.
- **Permalinks:** every "Real example" link uses a commit SHA (`/blob/<sha>/`), never `main`. Branch links rot.
- **Line wrapping:** prose is unwrapped (one sentence per line is fine — easier to diff). Snippets respect their own ecosystem's conventions.
- **Headings:** sentence case throughout. No ALL CAPS section labels.

## PR checklist

Before requesting review:

- [ ] `npx -y Ozark-Security-Labs/deterministic-deps@v1` passes locally at `low` severity threshold
- [ ] `npx markdownlint-cli2 "**/*.md"` is clean
- [ ] `lychee 'docs/**/*.md' 'README.md' 'CONTRIBUTING.md'` finds no broken links
- [ ] Every snippet you added has a corresponding file in `/examples/` (or extends one) and is referenced by relative link
- [ ] Every "Real example" link uses a commit-SHA permalink, not `main`
- [ ] The repo CI is green on your branch

The dogfooding CI runs all of the above automatically — failing builds block merge.

## Proposing a brand-new tier or appendix

Open a discussion (or issue) first. New tiers are rare — the three-tier structure is load-bearing for the doc's pedagogy. New appendices are easier, but should still scope to a single coherent topic.

## Code of conduct

Be the kind of reviewer and contributor you'd want to deal with. Substantive disagreement is welcome; condescension and pile-ons are not. Maintainers reserve the right to lock or remove threads that stop being useful.

## License of contributions

By submitting a contribution, you agree it is licensed under the same terms as the rest of the repository: [CC BY 4.0](LICENSE) for prose and [MIT](LICENSE-CODE) for code, configuration, and YAML examples.
