# Phase 1 — Engine core (proof)

Exit gate (checked in Session 1.6): properties P1–P5, P8, P9, P12 pass at `FC_NUM_RUNS=5000`; H1–H8 golden **explanation** expectations pass; S-dataset bench meets doc 03 targets for `isBookable` and `evaluate` without alternatives. If the gate fails, fix the engine before starting Phase 2. No UI work happens until this gate passes.

All Phase 1–2 sessions: use Codex reasoning effort **high**.

---

## Session 1.1 — Contracts

**Goal:** all engine-facing types exist as zod schemas with inferred TS types, so every later session builds against a fixed contract.

**Depends on:** 0.2
**Read first:** `docs/02-domain-and-rules.md` (Core concepts, Entities), `docs/03-engine-algorithms.md` (Public API), `docs/04-explanation-and-ai.md` (Explanation contract).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build** (`packages/contracts/src/`):
- `primitives.ts`: `Instant` (int, epoch minutes), `IsoDateTime` (string with offset), `LocalTime` (`HH:mm`), `LocalDate` (`YYYY-MM-DD`), `WeeklyHours`, `Window`, ID schemas (branded strings: `PractitionerId`, `RoomId`, `EquipmentId`, `ServiceId`, `PatientId`, `BookingId`, `PolicyVersionId`, `FactId`, `RuleId` enum of all rule IDs from doc 02).
- `snapshot.ts`: `Clinic`, `Practitioner`, `TimeOff`, `Room`, `Equipment`, `Service`, `Patient`, `Booking`, `Policy`, `ClinicSnapshot` (`{ clinic, practitioners, rooms, equipment, services, patients, bookings, policy, window: Window }`).
- `request.ts`: `BookingRequest`.
- `evaluation.ts`: `Fact`, `FactGroup`, `Explanation`, `Assignment` (`{ practitionerId, roomId, equipmentId: EquipmentId | null, start, end }`), `Alternative`, `AdminFix`, `ValidationError`, `Meta`, `Evaluation` (discriminated union on `outcome`), `Trace`, `CostWeights`, `EvaluateOptions`, `Audience`, `EXPLANATION_SCHEMA_VERSION = 1`.
- `index.ts` re-exports. No `api.ts` yet (Session 3.3).

**Tests:** schema accept/reject cases for each top-level schema (valid sample parses; missing field, wrong type, non-integer instant, bad `HH:mm`, unknown rule ID rejected).

**Done when:**
- [ ] Every type named in docs 02–04 contracts exists with matching fields (list them in the hand-off note).
- [ ] `RuleId` enum equals the doc 02 catalog exactly (22 IDs).
- [ ] `pnpm verify` green.

**Out of scope:** any logic; API schemas; fixtures.

**Starter prompt:**
```
$whynot-session
Session 1.1 — Contracts.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.1") fully before writing code.
Also read: docs/02-domain-and-rules.md, docs/03-engine-algorithms.md (Public API), docs/04-explanation-and-ai.md (Explanation contract).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.1-contracts. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 1.2 — Time module

**Goal:** one small, bullet-proof module for all wall-clock ↔ instant logic, so no other engine file ever touches timezones.

**Depends on:** 1.1
**Read first:** `docs/01-architecture.md` (Cross-cutting: Time), `docs/03-engine-algorithms.md` (Buffer semantics).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build** (`packages/engine/src/time.ts`, the only file importing luxon):
- `isoToInstant(iso)`, `instantToIso(instant, tz)`, `localDateOf(instant, tz)`, `localTimeOf(instant, tz)`, `weekdayOf(instant, tz)` (1=Mon…7=Sun).
- `localToInstant(date: LocalDate, time: LocalTime, tz)`; behaviour on DST gaps (nonexistent local time → next valid minute) and overlaps (ambiguous → earlier offset) documented in TSDoc and tested.
- `weeklyHoursToWindows(hours: WeeklyHours, date: LocalDate, tz): Window[]`.
- Interval helpers on half-open `[from, to)`: `overlaps`, `contains`, `gapBetween`, `mergeWindows`, `subtractWindows`.
- `addDays(date, n)`, `startOfLocalDay(date, tz)`, `durationMin`.

**Tests:** `Europe/London` spring-forward and fall-back days (opening hours across 01:00–02:00), `Asia/Kolkata` non-DST, midnight boundaries, exact-adjacency (`[a,b)` and `[b,c)` don't overlap), merge/subtract edge cases; property: `localToInstant ∘ (localDateOf, localTimeOf)` round-trips for valid local times.

**Done when:**
- [ ] All functions implemented with TSDoc; coverage of `time.ts` ≥ 95% lines.
- [ ] Lint confirms no other engine file imports luxon.
- [ ] `pnpm verify` green.

**Out of scope:** rules; formatting for display (narrator does that later).

**Starter prompt:**
```
$whynot-session
Session 1.2 — Time module.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.2") fully before writing code.
Also read: docs/01-architecture.md (Cross-cutting concerns), docs/03-engine-algorithms.md (Buffer semantics).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.2-time. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 1.3 — Rule catalog + global, patient & timing rules

**Goal:** the rule registry exists, and every rule that applies to the request as a whole (not to a specific resource) produces correct grounded facts.

**Depends on:** 1.2
**Read first:** `docs/02-domain-and-rules.md` (Rule catalog — all of it), `docs/03-engine-algorithms.md` (Steps 1 and 3).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build** (`packages/engine/src/`):
- `catalog.ts`: entry for **all 22 rules** with id, version (1), family, kind, weight, visibility, relaxableBy exactly as doc 02 (resource rules are registered now but implemented in 1.4).
- `factId.ts`: `factId(ruleId, parts: Record<string,string>)` → `"<ruleId>:k1=v1:k2=v2"` with keys sorted.
- `context.ts`: `EvalContext` built once per evaluation: canonicalised (sorted) snapshot, service, patient, requested interval, `now`, tz, per-day helper caches.
- `validate.ts`: Step 1 request validation → `ValidationError[]` (unknown ids, off-grid start, outside snapshot window).
- `rules/global.ts`: `clinic-closed-day`, `clinic-hours`, `min-notice`, `max-advance`.
- `rules/patient.ts`: `patient-eligibility-returning`, `patient-eligibility-referral`, `patient-daily-limit` (counts confirmed bookings of the patient on that local date; cancelled ignored).
- Each check returns `Fact[]` with complete `params` (local `HH:mm`, minutes, names) per doc 04.

**Tests:** per rule — violated, not violated, exact boundary (e.g. notice exactly equal = allowed), cancelled bookings ignored; catalog test asserting 22 entries matching a table literal copied from doc 02.

**Done when:**
- [ ] Catalog matches doc 02 exactly (test).
- [ ] 7 rules implemented with ≥ 3 tests each.
- [ ] `pnpm verify` green.

**Out of scope:** practitioner/room/equipment rules; candidates; explanation.

**Starter prompt:**
```
$whynot-session
Session 1.3 — Rule catalog + global, patient & timing rules.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.3") fully before writing code.
Also read: docs/02-domain-and-rules.md (entire rule catalog), docs/03-engine-algorithms.md (Steps 1 and 3), docs/04-explanation-and-ai.md (Fact).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.3-global-rules. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 1.4 — Practitioner, room & equipment rules

**Goal:** every resource-level rule produces correct grounded facts, including the precise buffer/turnover semantics.

**Depends on:** 1.3
**Read first:** `docs/02-domain-and-rules.md` (Rule catalog, notes), `docs/03-engine-algorithms.md` (Step 3, Buffer semantics).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build** (`packages/engine/src/rules/`):
- `practitioner.ts`: `practitioner-skill`, `practitioner-hours`, `practitioner-break`, `practitioner-time-off` (reason kept in params; visibility staff), `practitioner-busy` (per booking), `practitioner-buffer` (per booking; exact doc 03 semantics, using the *service of each booking* for its buffer), `practitioner-daily-cap` (policy default or per-practitioner cap), `request-practitioner` (single request-level fact).
- `room.ts`: `room-capability`, `room-maintenance`, `room-busy`, `room-turnover`.
- `equipment.ts`: `equipment-type`, `equipment-maintenance`, `equipment-busy`.
- Per-resource interval index (`src/index/intervals.ts`): bookings per practitioner/room/equipment sorted by start; overlap & adjacency lookup by binary search.
- Each family exposes `factsFor<Resource>(ctx, resourceId): Fact[]` — computed once per resource (factorisation).

**Tests:** per rule — violated / not / boundary; buffer: starting exactly at `be + buffer` allowed, one minute earlier blocked, our buffer running into next booking blocked, direct overlap yields `busy` not `buffer`; turnover same; daily cap exactly reached blocks, one below allows; cancelled bookings ignored everywhere.

**Done when:**
- [ ] All 15 resource rules implemented with ≥ 3 tests each; buffer/turnover have ≥ 6 each.
- [ ] Interval index has its own tests.
- [ ] `pnpm verify` green.

**Out of scope:** candidates/matrix; explanation; fixtures beyond inline test builders.

**Starter prompt:**
```
$whynot-session
Session 1.4 — Practitioner, room & equipment rules.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.4") fully before writing code.
Also read: docs/02-domain-and-rules.md (Rule catalog + notes), docs/03-engine-algorithms.md (Step 3, Buffer semantics).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.4-resource-rules. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 1.5 — Candidates, violation matrix, feasibility, reference implementation

**Goal:** `isBookable` and the violation matrix work, and are proven equal to an independent naive implementation on thousands of random clinics.

**Depends on:** 1.4
**Read first:** `docs/03-engine-algorithms.md` (Steps 2–3), `docs/06-testing-and-accuracy.md` (Properties, Reference implementation).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build:**
- `packages/engine/src/candidates.ts`: canonical enumeration (practitioners × rooms × equipment-or-null).
- `packages/engine/src/matrix.ts`: `buildMatrix(ctx)` → `{ candidates, factsById, V: FactIndexSet[] }` using `G ∪ P(p) ∪ R(r) ∪ E(e)`; bitset representation (`src/bitset.ts`).
- `packages/engine/src/evaluate.ts` (partial): `isBookable(snapshot, request, now)` with early exit and documented assignment preference order (requested practitioner, then lowest practitioner load that day, then ID). `evaluate` returns `bookable` or a temporary `blocked` with `explanation` **not yet populated** — expose only via an internal function; public `evaluate` lands in 1.6.
- `packages/engine/src/reference/feasible.ts`: naive, independent checker returning a valid assignment or null, plus `violations(assignment)`.
- `packages/fixtures/src/generate.ts`: fast-check arbitraries for valid snapshots and requests (sizes per doc 06), deterministic builders, `deepFreeze`.

**Tests:** P1, P2, P9 (property); differential test (matrix row empty ⇔ reference says that candidate valid); unit tests for assignment preference order.

**Done when:**
- [ ] P1, P2, P9 + differential pass at `FC_NUM_RUNS=5000`.
- [ ] Reference implementation is < 250 lines and contains no shared helpers with the optimised path except `time.ts` and contracts.
- [ ] `pnpm verify` green.

**Out of scope:** hitting set; alternatives; Riverside fixtures.

**Starter prompt:**
```
$whynot-session
Session 1.5 — Candidates, violation matrix, feasibility, reference implementation.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.5") fully before writing code.
Also read: docs/03-engine-algorithms.md (Steps 2–3), docs/06-testing-and-accuracy.md.
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.5-matrix. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 1.6 — Explanation, Riverside fixtures, hero goldens, benchmark → Phase 1 gate

**Goal:** blocked requests return a proven sufficient, minimal, minimum-weight explanation; the demo clinic exists; the hero stories are locked in as goldens; speed is measured.

**Depends on:** 1.5
**Read first:** `docs/03-engine-algorithms.md` (Step 4, Performance plan), `docs/02-domain-and-rules.md` (Riverside clinic, Hero scenarios), `docs/06-testing-and-accuracy.md` (Golden scenarios, Performance).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build:**
- `packages/engine/src/explain.ts`: reduce → forced → branch-and-bound (node cap 50,000, lower bound per doc) → greedy + prune fallback (`optimal: false`); internal sufficiency/irredundancy assertion; `perCandidateSummary`.
- Public `evaluate(snapshot, request, opts)` returning `bookable` / `blocked` (explanation populated; `alternatives: []`, `adminFix: null` until 2.1/2.2; `groups: []` until 2.2) / `invalid_request`, with `meta` (engineVersion `0.1.0`, timings).
- `packages/fixtures/src/clinic.ts`: deterministic Riverside Family Clinic per doc 02 relative to `DEMO_DATE` (default `2026-11-09`, a Monday), seeded PRNG booking load (~70%), plus hand-placed bookings/leave/maintenance for H1–H8 (IDs prefixed `bk-h<n>-`). It must not import fast-check (the browser bundles it in 4.4): export it as the subpath `@whynot/fixtures/clinic`, separate from `generate.ts`.
- `packages/fixtures/scenarios/H1…H8-*.json` with `outcome`, `primaryRuleIds`, `primaryFactIdsInclude` **only** (alternatives/adminFix/text expectations are added in later sessions from the doc 02 table). Write them from doc 02 **before** running the engine on them.
- `packages/engine/test/golden.test.ts` auto-discovering scenarios.
- `packages/engine/bench/`: datasets S/M/L, `pnpm bench` script, `baseline.json`.

**Tests:** P3, P4, P5, P8, P12 (property); explain unit tests (singleton forced, superset reduction, weight preference e.g. `min-notice` alone beats per-practitioner facts, greedy fallback triggered with a tiny node cap still irredundant); goldens H1–H8.

**Done when (Phase 1 gate):**
- [ ] P1–P5, P8, P9, P12 pass at `FC_NUM_RUNS=5000`.
- [ ] H1–H8 explanation goldens pass **without editing the expectations**. If one fails because the spec is wrong, stop and write it in `docs/QUESTIONS.md`.
- [ ] Bench S: `isBookable` p95 ≤ 2 ms, `evaluate` (no alternatives) p95 ≤ 20 ms; numbers recorded in hand-off and `baseline.json`.
- [ ] Gate result written in `docs/PROGRESS.md` (Phase 1 gate: PASS/FAIL + numbers).

**Out of scope:** alternatives, admin fix, grouping, projection, templates.

**Starter prompt:**
```
$whynot-session
Session 1.6 — Explanation, Riverside fixtures, hero goldens, benchmark (Phase 1 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-1-engine-core.md (section "Session 1.6") fully before writing code.
Also read: docs/03-engine-algorithms.md (Step 4, Performance plan), docs/02-domain-and-rules.md (Riverside clinic, Hero scenarios), docs/06-testing-and-accuracy.md.
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/1.6-explain. Implement exactly the "Build" list; nothing from "Out of scope".
Write the H1–H8 golden files from docs/02 BEFORE running the engine on them, and never edit their expectations to pass.
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 1 gate result) and opening a PR with $yeet.
```
