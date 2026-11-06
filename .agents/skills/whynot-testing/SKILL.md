---
name: whynot-testing
description: Use whenever writing or changing WhyNot tests — unit, fast-check property tests, differential tests against the reference implementation, golden scenario JSON files, PGlite integration tests, Playwright e2e and accessibility checks. Also use when a test fails and you are deciding how to fix it.
---

# Testing in WhyNot

Spec: `docs/06-testing-and-accuracy.md` (properties P1–P12, golden format, layers).

## Golden rule

A failing test means the **code** is wrong until proven otherwise. Never edit a golden expectation, weaken an assertion, lower `FC_NUM_RUNS`, add `.skip`/`.only`, or delete a test to get green. If you believe an expectation is wrong, stop and write the reasoning in `docs/QUESTIONS.md`.

## Where tests go

| Kind | Location | Naming |
| --- | --- | --- |
| Engine unit | `packages/engine/test/<module>.test.ts` | `describe('<rule or fn>')`, `it('<behaviour> when <condition>')` |
| Properties | `packages/engine/test/properties.test.ts` | one `it('P<n>: <name>')` per property |
| Differential | `packages/engine/test/differential.test.ts` | |
| Golden | `packages/fixtures/scenarios/<ID>-<slug>.json` + `packages/engine/test/golden.test.ts` (auto-discovers files) | |
| API integration | `apps/api/test/<route>.test.ts` (PGlite + `app.inject`) | `@pg` in the name if it needs real Postgres |
| E2E | `apps/web/e2e/<journey>.spec.ts` | one hero scenario per spec where possible |

## Property tests (fast-check)

- Use generators from `packages/fixtures/src/generate.ts`; extend them there, not inline.
- Runs: `numRuns: Number(process.env.FC_NUM_RUNS ?? 500)` (CI sets 5000). Always log the seed on failure (fast-check does by default — don't suppress).
- When a property fails: copy the shrunk counterexample into a named unit test, fix the code, keep the unit test.
- Deep-freeze generated inputs (`deepFreeze` helper) to catch mutation.

## Boundaries to always test for time logic

Exact start/end equality (half-open intervals `[start, end)`), midnight, DST transition days in `Europe/London` (last Sunday of March / October), slot grid edges, min-notice exactly equal, max-advance exactly equal, buffer exactly ending at start.

## Integration tests

- Build the app with a fresh PGlite DB per test file (`createTestApp()` helper from Session 3.1), migrate, seed only what the test needs.
- Freeze the clock with `DEMO_NOW`.
- Assert response bodies against the contract schemas (`schema.parse(body)`), plus specific fields.
- Never mock the engine.

## E2E

- Run against the built app (`pnpm build && pnpm test:e2e`); static-mode specs run against the static build.
- Use role/label selectors (`getByRole`, `getByLabel`), never CSS classes.
- Every page spec includes an axe check: no `serious` or `critical` violations.
