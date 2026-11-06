# Phase 6 — Staff tools

Exit gate (checked in Session 6.4): staff explanation view + admin fix visible for H1/H4/H5; insights dashboard populated from seed history; simulator result for "procedure buffer 15 → 10 min" matches a hand-checked count on a small fixture; inspector renders the H1 matrix; e2e + axe green for all staff pages.

---

## Session 6.1 — Staff explanation view, admin fix, policy editor

**Goal:** staff see the full picture — staff-only reasons, per-practitioner breakdown and the minimal policy fix — and can change policy safely.

**Depends on:** 5.3
**Read first:** `docs/03-engine-algorithms.md` (Step 6), `docs/04-explanation-and-ai.md` (grouping, staff templates), `docs/05-api-and-data.md` (policies).
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max`, `playwright`.

**Build:**
- Staff mode of the Why-not panel: staff wording, per-practitioner table (`perCandidateSummary`), "Policy fix" card listing concrete `changes` with a "Try in simulator" link, link "Open in inspector".
- `src/pages/staff/Policies.tsx`: current policy, version history, edit form → creates a new version (confirmation dialog explains effect); validation from contracts.

**Tests:** e2e: staff H1 shows Dr. Mehta's leave and the buffer fix; H5 shows cap fix; editing min-notice to 30 makes H3 bookable in patient view; axe on new screens.

**Done when:** e2e + axe green; screenshots in PR; `pnpm verify` green.

**Out of scope:** insights charts, simulator logic, inspector.

**Starter prompt:**
```
$whynot-session
Session 6.1 — Staff explanation view, admin fix, policy editor.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-6-staff-tools.md (section "Session 6.1") fully before writing code.
Also read: docs/03-engine-algorithms.md (Step 6), docs/04-explanation-and-ai.md, docs/05-api-and-data.md, docs/design/design-system.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max, playwright.
Work on branch session/6.1-staff-view. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 6.2 — Insights dashboard

**Goal:** a manager sees which rules cost the clinic bookings, at a glance.

**Depends on:** 6.1
**Read first:** `docs/05-api-and-data.md` (Insights queries), `docs/design/design-system.md` (chart styles).
**Skills:** `whynot-session`, `whynot-api`, `whynot-ui`, `ui-ux-pro-max`.

**Build:**
- `GET /api/admin/insights` (contract + SQL over `evaluations`): top blocking rules, blocked share per day, near misses grouped by admin-fix setting; filters: date range, service, practitioner.
- `src/pages/staff/Insights.tsx`: KPI tiles (requests, blocked %, top rule), bar chart of blocking rules (human labels from templates), daily line, "near misses" table with "Simulate this change" buttons. Charts accessible (data table toggle, labels, no colour-only encoding).

**Tests:** API integration on seeded DB with expected aggregates for a small hand-built evaluation set; component tests for empty/error states; e2e + axe.

**Done when:** aggregates test green; dashboard renders seeded data; screenshots; `pnpm verify` green.

**Out of scope:** simulator computation.

**Starter prompt:**
```
$whynot-session
Session 6.2 — Insights dashboard.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-6-staff-tools.md (section "Session 6.2") fully before writing code.
Also read: docs/05-api-and-data.md (Insights queries), docs/design/design-system.md.
Use skills: whynot-session, whynot-api, whynot-ui, ui-ux-pro-max.
Work on branch session/6.2-insights. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 6.3 — Policy simulator

**Goal:** "what if we shortened the buffer to 10 minutes?" answered with an honest estimate.

**Depends on:** 6.2
**Read first:** `docs/05-api-and-data.md` (Policy simulator), `docs/00-product-brief.md` (Honest limitations).
**Skills:** `whynot-session`, `whynot-api`, `whynot-engine`, `whynot-ui`.

**Build:**
- `packages/engine/src/simulate.ts` (pure): `applyPolicyChanges(snapshot, changes)` producing a modified snapshot; validated change types (policy fields, service buffer/turnover, practitioner cap, service eligibility).
- `POST /api/admin/simulate`: replay ≤ 2,000 historical requests against historical snapshots with changes; result per doc 05.
- `src/pages/staff/Simulator.tsx`: choose changes (pre-filled from insights/admin-fix links), run, show before/after, rules affected, example requests, and the permanent estimate disclaimer.

**Tests:** unit tests for `applyPolicyChanges`; integration test on a small hand-built history where the expected newly-bookable count is computed by hand and written in the test (document the hand calculation in a comment); performance: demo history simulation < 3 s.

**Done when:** hand-checked count matches; disclaimer visible; e2e + axe; `pnpm verify` green.

**Out of scope:** applying the change (that's the policy editor), knock-on modelling.

**Starter prompt:**
```
$whynot-session
Session 6.3 — Policy simulator.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-6-staff-tools.md (section "Session 6.3") fully before writing code.
Also read: docs/05-api-and-data.md (Policy simulator), docs/00-product-brief.md (Honest limitations).
Use skills: whynot-session, whynot-api, whynot-engine, whynot-ui.
Work on branch session/6.3-simulator. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 6.4 — Engine inspector → Phase 6 gate

**Goal:** a "show your work" view that makes the algorithm visible — the portfolio's technical wow moment.

**Depends on:** 6.3
**Read first:** `docs/03-engine-algorithms.md` (Steps 3, 4, 7).
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max`, `playwright`.

**Build:**
- `src/pages/staff/Inspector.tsx` using `/api/admin/evaluations/:id/trace`: candidate × fact matrix (rows = candidates, columns = facts, cells marked), highlighted chosen explanation columns, reduction step (supersets removed), node count / `optimal`, alternatives evaluated list with costs.
- Large matrices: virtualised or collapsible by practitioner; accessible table semantics.
- In static mode: inspector works for goldens (trace computed in-browser).

**Tests:** e2e: H1 inspector shows the buffer and leave columns highlighted; axe.

**Done when (Phase 6 gate):** all staff e2e + axe green; gate checklist at top of this file satisfied; result written in `docs/PROGRESS.md`.

**Out of scope:** editing from the inspector.

**Starter prompt:**
```
$whynot-session
Session 6.4 — Engine inspector (Phase 6 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-6-staff-tools.md (section "Session 6.4") fully before writing code.
Also read: docs/03-engine-algorithms.md (Steps 3, 4, 7), docs/design/design-system.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max, playwright.
Work on branch session/6.4-inspector. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 6 gate result) and opening a PR with $yeet (attach screenshots).
```
