# Phase 4 — Patient experience (the demo)

Exit gate (checked in Session 4.5): Playwright e2e for H1, H2, H3, H7 pass in **full** and **static** mode; axe reports no serious/critical issues; desktop + mobile screenshots approved by you; static build deploys to GitHub Pages.

Design quality matters here — this is what reviewers see first. Use `ui-ux-pro-max` throughout.

---

## Session 4.1 — Design system + web scaffold + API client + role switcher

**Goal:** a deliberate visual identity and a solid app shell, so every later screen is consistent.

**Depends on:** 3.4
**Read first:** `docs/01-architecture.md` (web), `docs/00-product-brief.md` (users, tone), `docs/09-codex-workflow.md` (UI UX Pro Max install), `.agents/skills/whynot-ui/SKILL.md`.
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max`.

**Build:**
- Install UI UX Pro Max into the repo (`npm install -g ui-ux-pro-max-cli`, `uipro init --ai codex`); commit its skill folder; note the install path in `AGENTS.md` Skills section.
- Run its design-system generator for: "healthcare clinic patient booking + staff analytics, calm, trustworthy, accessible, React + Tailwind". Save the result as `docs/design/design-system.md` (palette with contrast ratios, typography pair, spacing scale, radii, shadows, iconography, motion, component styles, chart styles). Resolve Q-3 by recording the chosen direction as D-045.
- `apps/web`: Tailwind configured from the design tokens; React Router routes (`/` booking, `/staff`, `/staff/insights`, `/staff/simulator`, `/staff/inspector/:id` placeholders); TanStack Query provider; `src/api/client.ts` (typed fetch + zod parse, `X-Demo-Role` headers) and hooks for `clinic`, `availability`, `explain`.
- App shell: header with "Riverside Family Clinic · Demo — fictional data", demo clock, role switcher (Patient: pick fictional patient / Staff); footer link to GitHub + "How it works".
- Base components in `src/components/ui/`: Button, Card, Badge, Dialog/Drawer, Select, Tabs, Skeleton, EmptyState, ErrorState, Toast — styled from tokens, accessible.
- `apps/web/AGENTS.md`: area rules pointing to `$whynot-ui` and `docs/design/design-system.md`.

**Tests:** component tests (Vitest + Testing Library) for Button/Dialog focus behaviour; API client parses and rejects malformed responses.

**Done when:**
- [ ] `docs/design/design-system.md` exists and D-045 recorded.
- [ ] Shell runs against the local API; role switching changes headers.
- [ ] Screenshots (1280 px, 390 px) in the PR.
- [ ] `pnpm verify` green.

**Out of scope:** booking page content, Why-not panel, static mode.

**Starter prompt:**
```
$whynot-session
Session 4.1 — Design system + web scaffold + API client + role switcher.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-4-patient-ui.md (section "Session 4.1") fully before writing code.
Also read: docs/00-product-brief.md, docs/01-architecture.md, docs/09-codex-workflow.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max (install it first as described).
Work on branch session/4.1-design-shell. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 4.2 — Booking page + availability grid

**Goal:** a patient can choose a service, optionally a practitioner, and a day, and see a clear grid of bookable and unavailable times.

**Depends on:** 4.1
**Read first:** `docs/05-api-and-data.md` (`/clinic`, `/availability`), `docs/design/design-system.md`.
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max`.

**Build:**
- `src/pages/Booking.tsx`: step layout (Service → Practitioner "Any" default → Day picker over the demo fortnight → time grid).
- `src/components/AvailabilityGrid.tsx`: slots as buttons; bookable vs unavailable styled with icon + text (not colour only); accessible names per `$whynot-ui`; unavailable slots are clickable ("See why").
- Loading / empty ("No sessions offered this day") / error states.
- URL state (`?service=&practitioner=&date=`) so scenarios are linkable.

**Tests:** component tests for grid accessibility names and keyboard navigation; e2e smoke (page loads, grid renders for H1 day).

**Done when:**
- [ ] Grid matches `/availability` for H1's day (e2e assertion).
- [ ] Keyboard-only flow works; screenshots in PR.
- [ ] `pnpm verify` green.

**Out of scope:** Why-not panel, booking action.

**Starter prompt:**
```
$whynot-session
Session 4.2 — Booking page + availability grid.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-4-patient-ui.md (section "Session 4.2") fully before writing code.
Also read: docs/05-api-and-data.md, docs/design/design-system.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max.
Work on branch session/4.2-booking-page. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 4.3 — "Why not?" panel, alternatives, one-click booking

**Goal:** the signature moment — click a blocked slot, understand why in seconds, book an alternative in one click.

**Depends on:** 4.2
**Read first:** `docs/04-explanation-and-ai.md` (grouping, templates), `docs/05-api-and-data.md` (`/explain`, `/bookings`, 409 flow), `docs/02-domain-and-rules.md` (Hero scenarios).
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max`, `playwright`.

**Build:**
- `src/components/WhyNotPanel.tsx` (drawer on desktop, bottom sheet on mobile): headline ("Tuesday 14:30 isn't available"), grouped reasons rendered with **template** text from `@whynot/narrator/templates` (icon per kind), capability-context line, alternatives as large buttons with what changes ("15 min later · same doctor"), "no alternatives" state (incl. H4 copy).
- Booking a slot or an alternative → `POST /api/bookings` → success confirmation; on 409 → panel refreshes with the new explanation ("Just taken — here's why").
- Wording-source label ("Plain wording") reserved for Phase 5 toggle.
- "How was this decided?" disclosure: one-paragraph explanation of the engine + link to the write-up (placeholder).

**Tests:** e2e H1 (reasons include buffer + unavailable Dr. Mehta, top alternative 14:45 Dr. Rao, book it → confirmed), H3 (notice), H7 (preference trade-off shows both alternatives); component tests for focus management and `aria-live`.

**Done when:**
- [ ] H1, H3, H7 e2e green; the H1 flow takes ≤ 3 clicks from grid to confirmed alternative.
- [ ] Screenshots of H1 panel (desktop + mobile) in PR; you approve them.
- [ ] `pnpm verify` green.

**Out of scope:** AI wording, staff view, static mode.

**Starter prompt:**
```
$whynot-session
Session 4.3 — "Why not?" panel, alternatives, one-click booking.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-4-patient-ui.md (section "Session 4.3") fully before writing code.
Also read: docs/04-explanation-and-ai.md, docs/05-api-and-data.md, docs/02-domain-and-rules.md (Hero scenarios), docs/design/design-system.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max, playwright.
Work on branch session/4.3-why-not-panel. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet (attach screenshots).
```

---

## Session 4.4 — Static demo mode + GitHub Pages

**Goal:** the full patient demo runs in the browser with no backend, deployable for free.

**Depends on:** 4.3
**Read first:** `docs/01-architecture.md` (Runtime modes), `.agents/skills/whynot-ui/SKILL.md` (static mode rule).
**Skills:** `whynot-session`, `whynot-ui`.

**Build:**
- `src/api/adapter.ts`: interface used by hooks; `httpAdapter` (existing) and `src/static/staticAdapter.ts` (builds the Riverside snapshot from `@whynot/fixtures` in-browser, calls `evaluate`/`isBookable`/`projectForAudience`; bookings held in memory + `localStorage`, "Reset demo" button).
- Build flag `VITE_MODE=static`; banner "Static demo — runs entirely in your browser".
- Staff routes in static mode show a friendly "Run the full version to see staff tools" card (until Phase 6 adds static support where cheap).
- `.github/workflows/pages.yml`: build static web and deploy to GitHub Pages on push to `main` (with correct `base` path).
- Bundle check: engine + fixtures add ≤ 250 KB gzipped (record actual).

**Tests:** static-mode e2e for H1 and H2 (run against `vite preview` of the static build); unit test that static and http adapters return deep-equal projected evaluations for all goldens.

**Done when:**
- [ ] Static e2e green; adapter equality test green.
- [ ] Pages workflow succeeds; public URL recorded in README and hand-off.
- [ ] `pnpm verify` green.

**Out of scope:** AI narrations in static mode (5.3), staff tools.

**Starter prompt:**
```
$whynot-session
Session 4.4 — Static demo mode + GitHub Pages.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-4-patient-ui.md (section "Session 4.4") fully before writing code.
Also read: docs/01-architecture.md (Runtime modes).
Use skills: whynot-session, whynot-ui.
Work on branch session/4.4-static-mode. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 4.5 — E2E, accessibility, responsive polish → Phase 4 gate

**Goal:** the patient demo is robust, accessible and polished on every screen size.

**Depends on:** 4.4
**Read first:** `docs/06-testing-and-accuracy.md` (E2E, Accessibility), `docs/design/design-system.md`.
**Skills:** `whynot-session`, `whynot-ui`, `ui-ux-pro-max` (pre-delivery checklist), `playwright`, `playwright-interactive`, `screenshot`.

**Build:**
- Wire `pnpm test:e2e` for real (full mode starts API on PGlite + `DEMO_NOW`; static mode uses preview build); add e2e job to CI.
- Add H2 full-mode and H1/H3/H7 static-mode specs; axe check on every page.
- Fix everything from the `ui-ux-pro-max` pre-delivery checklist (contrast, focus states, hit targets ≥ 44 px, responsive at 360/390/768/1280 px, reduced-motion).
- Explore the app with `playwright-interactive` for broken states; fix them.
- Screenshots of key screens saved in `docs/design/screens/`.

**Done when (Phase 4 gate):**
- [ ] E2E H1, H2, H3, H7 green in both modes in CI.
- [ ] axe: 0 serious/critical.
- [ ] You review the screenshots and approve.
- [ ] Gate result written in `docs/PROGRESS.md`.

**Out of scope:** new features.

**Starter prompt:**
```
$whynot-session
Session 4.5 — E2E, accessibility, responsive polish (Phase 4 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-4-patient-ui.md (section "Session 4.5") fully before writing code.
Also read: docs/06-testing-and-accuracy.md, docs/design/design-system.md.
Use skills: whynot-session, whynot-ui, ui-ux-pro-max, playwright, playwright-interactive, screenshot.
Work on branch session/4.5-polish. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 4 gate result) and opening a PR with $yeet (attach screenshots).
```
