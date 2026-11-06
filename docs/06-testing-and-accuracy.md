# 06 — Testing & accuracy

WhyNot's credibility is that its answers are **provably correct** and **judged helpful**. Those are different and are tested differently.

## The single gate: `pnpm verify`

Every session ends with `pnpm verify` green. It runs, in order:

1. `pnpm format:check` (Prettier)
2. `pnpm lint` (ESLint incl. import-boundary rules)
3. `pnpm typecheck` (tsc `--noEmit`, all packages, strict)
4. `pnpm test` (Vitest: unit + property + golden + integration on PGlite)
5. `pnpm build` (all packages and apps)

CI runs the same script, plus Playwright e2e and the `@pg` tests against a real Postgres service on `main` and PRs. A session is not done if `pnpm verify` is red — no exceptions, no skipped tests.

## Test layers

| Layer | Tool | Where | What it proves |
| --- | --- | --- | --- |
| Unit | Vitest | `*/test/*.test.ts` | Each rule check, time conversion, template, verifier check |
| **Property** | fast-check | `engine/test/properties.test.ts` | Engine invariants hold for thousands of random clinics and requests |
| **Differential** | Vitest + reference impl | `engine/test/differential.test.ts` | Optimised engine agrees with a naive, obviously-correct implementation |
| **Golden** | Vitest + JSON scenarios | `fixtures/scenarios/*.json` | Hero and edge scenarios produce the human-expected explanation and alternatives |
| Integration | Vitest + PGlite + Fastify `inject` | `apps/api/test` | Routes, validation, projection, booking transactions |
| E2E | Playwright | `apps/web/e2e` | Hero journeys in a real browser (full mode and static mode) |
| Accessibility | @axe-core/playwright | e2e | No serious/critical violations on key pages |
| Narration eval | `tools/eval-narration` | manual/CI-optional | AI wording quality and verifier pass rate |
| Benchmarks | Vitest bench / tinybench | `engine/bench` | Speed targets (doc 03) |

## Engine properties (must always hold)

Generators (`fixtures/src/generate.ts`) produce random but valid snapshots (1–8 practitioners, 1–6 rooms, 0–3 equipment units, random hours/breaks/leave/maintenance, 0–60 bookings, random policies) and random requests, with a fixed seed per CI run printed on failure.

| ID | Property |
| --- | --- |
| P1 | **Feasibility correctness:** `outcome === 'bookable'` ⇔ the reference implementation finds a valid assignment |
| P2 | **Bookable assignments are valid:** the returned assignment violates no rule (checked by the reference) |
| P3 | **Sufficiency:** every candidate's `V(a)` intersects `explanation.primary` |
| P4 | **Irredundancy (minimality):** for each primary fact `x`, some candidate has `V(a) ∩ primary = {x}` |
| P5 | **Optimality:** when `optimal === true`, no hitting set of lower weight exists (brute force on instances with ≤ 12 distinct facts) |
| P6 | **Alternatives are bookable:** every alternative re-evaluates to `bookable` with the same assignment |
| P7 | **Alternatives are distinct and ordered:** no duplicate (practitioner, start); cost non-decreasing within the diversity rules |
| P8 | **Determinism:** shuffling every input array yields a deep-equal result |
| P9 | **Purity:** calling twice with the same input yields equal output; input objects are not mutated (deep-frozen inputs in tests) |
| P10 | **Privacy:** patient projection contains no staff-only rule IDs, no `reason` params, no other patient IDs/names, no raw booking IDs |
| P11 | **Admin fix validity:** relaxing exactly the admin-fix facts makes the original request bookable; `adminFix` contains only admin-relaxable facts |
| P12 | **Fact integrity:** every `primary` id exists in `facts`; every fact's `ruleId` is in the catalog |

Run counts: 5,000 cases per property in CI (`FC_NUM_RUNS`), 500 locally by default for speed.

## Reference implementation

`engine/src/reference/` is a deliberately naive feasibility checker written **independently** of the optimised path: it loops over every practitioner × room × equipment combination and checks every rule directly from the snapshot, with no factorisation, indexes or bitsets. It must stay simple enough to review by eye. It is used only in tests.

## Golden scenarios

Format (`fixtures/scenarios/H1-procedure-buffer.json`):

```json
{
  "id": "H1",
  "title": "Minor procedure blocked by buffer + leave",
  "snapshot": "riverside",          // or inline overrides
  "now": "2026-11-09T09:00:00+00:00",
  "request": { "patientId": "pt-asha", "serviceId": "svc-minor-proc", "start": "2026-11-10T14:30:00+00:00" },
  "expect": {
    "outcome": "blocked",
    "primaryRuleIds": ["practitioner-buffer", "practitioner-time-off", "practitioner-skill"],
    "primaryFactIdsInclude": ["practitioner-buffer:prac=dr-rao:booking=bk-h1-a"],
    "topAlternative": { "start": "2026-11-10T14:45:00+00:00", "practitionerId": "dr-rao" },
    "adminFixSettings": ["service.svc-minor-proc.bufferAfterMin"],
    "patientTextIncludes": ["15-minute", "14:45"]
  }
}
```

Rules:

- Scenarios are written from the **spec (doc 02)**, before the code that satisfies them.
- An expectation may only be changed together with a decision-log entry explaining why the original expectation was wrong. Never "update the golden file to make it pass".
- Required set by v1.0: H1–H8 + at least 16 edge scenarios (DST change day, request at closing boundary, appointment spanning a break edge, back-to-back exact adjacency, zero buffers, all practitioners unqualified, no rooms with capability, patient limit + notice together, preference + leave, equipment-only conflict, maintenance spanning midnight, max-advance boundary, min-notice exact boundary, daily cap exactly reached, cancelled booking ignored, two equal-cost alternatives tie-break).

## Helpfulness review (human)

For each hero scenario, the reviewer (you) scores the patient and staff wording 1–5 on: **correct**, **clear**, **complete enough**, **kind**, **actionable**. Target: no score below 4 on hero scenarios. Results are kept in `docs/reviews/helpfulness-<date>.md`. This is the honest measure of "is it actually good", separate from correctness.

## Narration eval

`pnpm eval:narration` runs every golden scenario × both audiences through `OpenAINarrator` (needs `OPENAI_API_KEY`) and writes `reports/narration-<date>.md` with:

- verifier pass rate on first attempt (target ≥ 90% before shipping AI wording by default),
- failure reasons by check,
- privacy failures (target **0**; any privacy failure blocks enabling AI wording for patients),
- side-by-side template vs AI text for manual review,
- token usage and estimated cost.

It also writes `apps/web/src/static/narrations.json` for the static demo (only verified narrations).

## Performance tests

`pnpm bench` prints p50/p95 for the operations in doc 03 on datasets S/M/L. A lightweight CI check fails if S-dataset `evaluate` p95 regresses by > 50% versus the committed baseline (`engine/bench/baseline.json`).

## What "accurate" means in our claims

We will claim: *"For the rules it models, WhyNot's explanations are sufficient and minimal, and every suggested alternative is verified bookable — checked by property tests on thousands of generated clinics."*

We will **not** claim: that explanations match every human's intuition, that the AI wording is always perfect (it's verified or replaced), or that it reflects rules the clinic never configured.
