# Phase 2 — Alternatives, admin fix, privacy, wording

Exit gate (checked in Session 2.4): P6, P7, P10, P11 pass at 5,000 runs (plus all Phase 1 properties still pass); H1–H8 goldens fully pass including alternatives, admin fix and patient text; every catalog rule has both templates; ≥ 16 edge goldens pass; bench S `evaluate` with alternatives p95 ≤ 400 ms.

Reasoning effort: **high**.

---

## Session 2.1 — Alternatives search

**Goal:** blocked evaluations return up to 3 verified-bookable, diverse alternatives ranked by the documented cost model.

**Depends on:** 1.6
**Read first:** `docs/03-engine-algorithms.md` (Step 5), `docs/02-domain-and-rules.md` (Hero scenarios — alternatives column).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build:**
- `packages/engine/src/alternatives.ts`: lazy generators per category (`same-day`, `other-practitioner`, `other-day`), merged by cost; `CostWeights` defaults exactly as doc 03; skip times before `now + minNotice`; `budgetEvals` cap; diversity selection; `changes[]` and `cost` per alternative.
- Wire into `evaluate` (`opts.alternatives` defaults `{ max: 3, searchDays: 14, budgetEvals: 3000 }`).
- Add `topAlternative` expectations to H1–H8 goldens **exactly from the doc 02 table** (adding expectations is allowed; changing existing ones is not). H4 expects `alternatives: []`.

**Tests:** P6, P7 (property); unit: cost ordering, diversity (one per category when available), requested-practitioner penalty, earlier-vs-later tie-break, budget exhaustion returns partial list, no duplicates, none before min-notice.

**Done when:**
- [ ] P6, P7 pass at 5,000 runs; all Phase 1 properties still pass.
- [ ] H1–H8 `topAlternative` expectations pass.
- [ ] Bench S `evaluate` with alternatives p95 ≤ 400 ms (recorded).
- [ ] `pnpm verify` green.

**Out of scope:** admin fix; grouping; projection; UI.

**Starter prompt:**
```
$whynot-session
Session 2.1 — Alternatives search.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-2-engine-complete.md (section "Session 2.1") fully before writing code.
Also read: docs/03-engine-algorithms.md (Step 5), docs/02-domain-and-rules.md (Hero scenarios).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/2.1-alternatives. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 2.2 — Admin fix, presentation groups, final `evaluate()` shape

**Goal:** the engine output is complete: staff-only minimal policy fix with concrete setting changes, and deterministic presentation groups.

**Depends on:** 2.1
**Read first:** `docs/03-engine-algorithms.md` (Steps 6–7), `docs/04-explanation-and-ai.md` (Presentation grouping).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build:**
- `packages/engine/src/adminFix.ts`: candidates whose `V(a)` is fully admin-relaxable; minimum admin cost; concrete `changes` (`setting` path strings like `policy.minNoticeMin`, `service.<id>.bufferAfterMin`, `practitioner.<id>.dailyCap`, `service.<id>.eligibility`, with computed `to` values — e.g. the largest buffer that still fits).
- `packages/engine/src/groups.ts`: grouping rules from doc 04 (`request`, `per-practitioner`, `capability-context`, `resources`), deterministic order.
- `trace` option (Step 7) with size cap.
- `ENGINE_VERSION` → `0.2.0`; `evaluate` result now matches the full contract.
- Add `adminFixSettings` expectations to goldens from doc 02 (H4 eligibility, H5 daily cap, H1 buffer, H3 min notice).

**Tests:** P11 (property); unit: buffer `to` value computation, cap `to` = cap+1, no fix when blocked by occupancy only, grouping of H1 into practitioner + capability-context groups.

**Done when:**
- [ ] P11 passes at 5,000 runs; all earlier properties pass.
- [ ] Goldens with admin-fix expectations pass.
- [ ] Contract schema `Evaluation.parse(result)` succeeds for every golden and 1,000 random cases (test).
- [ ] `pnpm verify` green.

**Out of scope:** projection; templates; UI.

**Starter prompt:**
```
$whynot-session
Session 2.2 — Admin fix, presentation groups, final evaluate() shape.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-2-engine-complete.md (section "Session 2.2") fully before writing code.
Also read: docs/03-engine-algorithms.md (Steps 6–7), docs/04-explanation-and-ai.md (Presentation grouping).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/2.2-admin-fix. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 2.3 — Audience projection (privacy)

**Goal:** a single, proven function turns a staff evaluation into a patient-safe one.

**Depends on:** 2.2
**Read first:** `docs/04-explanation-and-ai.md` (Audience projection).
**Skills:** `whynot-session`, `whynot-engine`, `whynot-testing`.

**Build:**
- `packages/engine/src/project.ts`: `projectForAudience(evaluation, audience, { requesterPatientId })` implementing rules 1–5 of doc 04 (generic replacement facts, drop room/equipment capability unless none qualifies, remove adminFix/perCandidateSummary/trace, opaque per-evaluation fact IDs via a deterministic non-crypto hash of `(evaluationSalt, factId)` — salt passed in by the caller).
- Export from package index.

**Tests:** P10 (property, serialising the projection and scanning for forbidden strings: every booking ID, every other patient's ID/name, every `reason` value, staff-only rule IDs); unit tests for each rule; H1 patient projection shows Dr. Mehta as "unavailable".

**Done when:**
- [ ] P10 passes at 5,000 runs.
- [ ] Projection output parses against the `Evaluation` schema.
- [ ] `pnpm verify` green.

**Out of scope:** templates; API.

**Starter prompt:**
```
$whynot-session
Session 2.3 — Audience projection (privacy).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-2-engine-complete.md (section "Session 2.3") fully before writing code.
Also read: docs/04-explanation-and-ai.md (Audience projection).
Use skills: whynot-session, whynot-engine, whynot-testing.
Work on branch session/2.3-projection. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 2.4 — Template narrator + edge goldens → Phase 2 gate

**Goal:** every evaluation can be rendered as clear, kind patient text and precise staff text without any AI; the edge-case corpus is complete.

**Depends on:** 2.3
**Read first:** `docs/04-explanation-and-ai.md` (Template wording), `docs/06-testing-and-accuracy.md` (Golden scenarios — edge list).
**Skills:** `whynot-session`, `whynot-narration`, `whynot-testing`.

**Build:**
- `packages/narrator/src/format.ts`: time/date/duration formatting in clinic tz & locale (`en-GB` default).
- `packages/narrator/src/templates/`: one template per rule × audience (44) + group templates + alternatives sentence ("Instead, you could book: …") + "no alternatives" copy (incl. the H4 eligibility copy).
- `packages/narrator/src/templateNarrator.ts`: `TemplateNarrator` implementing the `Narrator` interface (doc 04) with `source: 'template'`; define the `Narrator`, `NarrationInput`, `Narration` types in contracts.
- Add `patientTextIncludes` expectations to H1–H8.
- ≥ 16 edge golden scenarios from the doc 06 list (`E01…E16`), written from spec first.

**Tests:** catalog-coverage test (every rule has both templates); snapshot tests of patient + staff text for H1–H8; edge goldens; Phase 1–2 properties all still green.

**Done when (Phase 2 gate):**
- [ ] All goldens (H1–H8 complete, E01–E16) pass without editing expectations.
- [ ] P1–P12 pass at 5,000 runs.
- [ ] You (the human) read the H1–H8 patient texts in the PR description and approve their tone.
- [ ] Gate result written in `docs/PROGRESS.md`.

**Out of scope:** OpenAI; verifier; UI.

**Starter prompt:**
```
$whynot-session
Session 2.4 — Template narrator + edge goldens (Phase 2 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-2-engine-complete.md (section "Session 2.4") fully before writing code.
Also read: docs/04-explanation-and-ai.md (Template wording, Narrator interface), docs/06-testing-and-accuracy.md (Golden scenarios).
Use skills: whynot-session, whynot-narration, whynot-testing.
Work on branch session/2.4-templates. Implement exactly the "Build" list; nothing from "Out of scope".
Write edge goldens from the spec before running the engine on them; never edit expectations to pass.
Paste the H1–H8 patient and staff texts into the PR description for human review.
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 2 gate result) and opening a PR with $yeet.
```
