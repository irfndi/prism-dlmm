---
name: prism-maintain
version: "0.1.0"
description: >
  Maintain the Prism codebase: measure cyclomatic complexity, refactor
  hotspots, and enforce the anti-slop type discipline. Use when the user asks
  about "complexity", "refactor", "hotspot", "maintainability", "code health",
  "anti-slop", "lint rules", or "clean up" the Prism repo.
license: MIT
author: irfndi
homepage: https://github.com/irfndi/prism-liquidity-agent
tags: [typescript, effect-ts, refactoring, complexity, lint, maintainability]
compatibility: Requires Bun 1.4+ and oxlint with the vendored anti-slop plugin.
---

# Prism Code Maintenance

Measure complexity, refactor hotspots, keep the code human-maintainable.
Methodology adapted from
[saurabhkumar8112/cyclomatic-complexity-skill](https://github.com/saurabhkumar8112/cyclomatic-complexity-skill)
(Apache-2.0); type discipline enforced by the vendored
[dmmulroy/anti-slop](https://github.com/dmmulroy/anti-slop) v0.1.2 plugin in
`tools/oxlint/anti-slop/`.

## Complexity gates (project config wins)

`bun run lint` enforces `complexity: max 10` as an error via oxlint
(type-aware — it is the authoritative measure for this Effect-TS codebase;
line-counters like lizard mis-parse `Effect.gen` chains and report phantom
CCN 50+ on 15-line functions, so never act on lizard numbers here).

Bands: 1–6 fine · 7–10 watch (refactor only if touching the function anyway) ·
11+ must split before merge (lint fails).

Watch-band probe (never commit this config):

```bash
sed '"'"'s/"max": 10/"max": 6/'"'"' .oxlintrc.json > .oxlint-cc6.json
bunx oxlint --config .oxlint-cc6.json engine ops bench cli
rm .oxlint-cc6.json
```

## Workflow (mandatory)

1. Measure the touched functions, rank by CCN descending.
2. Report hotspots with numbers BEFORE touching anything.
3. Refactor worst first, one function at a time.
4. Re-measure; close with the report table below.
5. Verify: `bun run lint`, `bun run format`, full `bun run test` green with
   ZERO test edits (unchanged tests passing is the behavior proof).

```md
## Complexity report
| Function | Before | After |
|----------|--------|-------|
| runPoolEntryGates | 11 | 9 |
Extracted: isSafetyBinsSkippable, snapshotPriceDrift
Behavior verified: full suite green, no test edits
```

## Refactor tactics (preference order)

1. Guard clauses — invert, return early, kill nesting.
2. Extract function — name for WHAT, not how.
3. Lookup table / map over if-else chains.
4. Named predicates — `isPositiveFiniteApr(x)` beats boolean soup.
5. Lookup before polymorphism: Effect code dispatches via Layers and
   tagged unions, not class hierarchies.
6. `continue` over nested `if` in loops.

Hard rules: preserve behavior · never game the metric (a dense one-liner
hiding 6 branches is worse than the honest chain) · never break exported
signatures without asking · one responsibility per function.

## When NOT to split

- Numbered risk-gate chains (`engine/risk-service.ts`): sequential
  early-return blocks are the documented convention (AGENTS.md) — splitting
  scatters the gate narrative.
- Single decision-loop `Effect.gen` blocks (`engine/program.ts`): splitting
  obscures per-cycle sequencing; extract pure predicates instead.
- Agent-overlay handlers and network retry logic in untouched domains.

## Anti-slop plugin (vendored v0.1.2 parity)

Source: `tools/oxlint/anti-slop/` (`.ts` sources + `index.mjs` bundles +
upstream `*.test.ts` contracts, which need `oxlint/plugins-dev` to execute).
Rebuild bundles after any source change — never hand-edit `.mjs`:

```bash
bun build tools/oxlint/anti-slop/index.ts --outfile tools/oxlint/anti-slop/index.mjs
bun build tools/oxlint/anti-slop/effect/index.ts --outfile tools/oxlint/anti-slop/effect/index.mjs
```

Rules that most often bite: `no-known-value-widening` (open-dict
annotations — prefer finite `type` contracts, `satisfies`, accumulators, or
`Map`), `require-safety-comment-for-type-assertion` (every `as` needs a
`SAFETY:` justification with non-empty text), `no-unknown-parameters` /
`no-unknown-returns` (parse at the boundary, return named domain types).
Never suppress or weaken a rule — fix the code.
