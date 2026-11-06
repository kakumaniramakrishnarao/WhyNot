---
name: whynot-engine
description: Use for any change in packages/engine or packages/contracts of WhyNot — rules, rule catalog, time handling, candidates, violation matrix, explanation (hitting set), alternatives, admin fix, audience projection. Not for API routes or UI.
---

# Working on the WhyNot engine

Spec: `docs/02-domain-and-rules.md`, `docs/03-engine-algorithms.md`, `docs/04-explanation-and-ai.md` (contract + projection). Follow them exactly.

## Purity checklist (every change)

- [ ] No imports except `@whynot/contracts` and `luxon` (luxon only in `src/time.ts`).
- [ ] No `Date`, `Date.now`, `Math.random`, `process`, `console`, timers, globals, I/O.
- [ ] Inputs are never mutated (tests deep-freeze them).
- [ ] All iteration over collections uses canonical order (sorted by `id`); tie-breaks by ID string compare (`a < b`), never by locale compare.
- [ ] Instants are integer epoch minutes; durations are integer minutes.

## Adding or changing a rule (end-to-end)

1. Catalog entry in `src/catalog.ts`: `id`, `version`, `family`, `kind`, `weight`, `visibility`, `relaxableBy` — values from doc 02's table. Changing semantics ⇒ bump `version`.
2. Check function in the right `src/rules/<family>.ts`. Signature returns grounded `Fact`s (with stable `id` built by `factId(ruleId, {key: value,...})` — keys sorted).
3. Fact `params` must contain every value the templates and verifier need (times as clinic-local `HH:mm`, durations, names). Nothing private in public-visibility params.
4. Add it to the reference implementation (`src/reference/`) in the naive style.
5. Templates (both audiences) in `packages/narrator/src/templates/` — the catalog-coverage test will fail otherwise.
6. Tests: unit (violated / not violated / boundary-exact cases), and make sure property generators can produce situations that trigger it.
7. A golden scenario that features the rule.

## Algorithms

- Matrix: compute families per resource once; `V(a) = G ∪ P(p) ∪ R(r) ∪ E(e)`.
- Explanation: reduce (dedupe + drop supersets) → forced singletons → exact branch-and-bound (node cap 50,000) → else greedy + prune-to-irredundant, `optimal: false`. Always run the internal sufficiency + irredundancy assertion before returning.
- Alternatives: lazy cost-ordered generators per category, `isBookable` fast path, `budgetEvals` cap, diversity selection. Every returned alternative must be re-evaluated bookable.
- Admin fix: candidates whose `V(a)` is entirely admin-relaxable; minimum admin cost; compute concrete setting changes.

## Performance

Run `pnpm bench` after algorithm changes. Don't regress S-dataset p95 by > 50% versus `engine/bench/baseline.json`. Update the baseline only when a session card says so.
