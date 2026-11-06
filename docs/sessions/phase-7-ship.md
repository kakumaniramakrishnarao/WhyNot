# Phase 7 — Ship

Exit gate: the v1.0 release checklist in `docs/07-roadmap.md` is 100% ticked.

---

## Session 7.1 — Hardening

**Goal:** close security, reliability and performance gaps before anyone else uses it.

**Depends on:** 6.4
**Read first:** `docs/security/threat-model.md` (draft from 3.3), `docs/06-testing-and-accuracy.md` (Performance), `AGENTS.md`.
**Skills:** `whynot-session`, `security-threat-model`, `security-best-practices`, `whynot-review`.

**Build:**
- Finalise `docs/security/threat-model.md`; run `security-best-practices` over the whole repo; fix findings or document accepted risks with reasons.
- Verify rate limits, body limits, CORS, headers (helmet-equivalent), error messages leak nothing, logs contain no PII, `.env.example` complete, dependency audit (`pnpm audit`) addressed.
- Error/empty states reviewed across all pages; API timeouts handled in UI.
- Re-run `pnpm bench`; confirm all doc 03 targets; update baseline.
- Run `$whynot-review` over the full codebase against AGENTS.md invariants (not just a diff); fix blockers/majors.

**Done when:** threat model final; no open blocker/major findings; bench targets met; `pnpm verify` + CI e2e green.

**Out of scope:** new features.

**Starter prompt:**
```
$whynot-session
Session 7.1 — Hardening.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-7-ship.md (section "Session 7.1") fully before writing code.
Also read: docs/security/threat-model.md, docs/06-testing-and-accuracy.md.
Use skills: whynot-session, security-threat-model, security-best-practices, whynot-review.
Work on branch session/7.1-hardening. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 7.2 — Deploy, environments, runbook

**Goal:** the static demo is live and stable; the full version runs with one command; optionally the full stack is hosted.

**Depends on:** 7.1
**Read first:** `docs/01-architecture.md` (Runtime modes), `docs/08-decisions.md` (D-035, Q-5).
**Skills:** `whynot-session`, plus one deploy skill if hosting is chosen (e.g. `render-deploy`, `vercel-deploy`, `netlify-deploy`, `cloudflare-deploy` from openai/skills curated).

**Build:**
- Confirm GitHub Pages static demo on `main` (custom 404, correct base path, social preview image).
- `docs/runbook.md`: run locally (`docker compose up -d`, `pnpm db:migrate`, `pnpm db:seed`, `pnpm dev`), env vars, reset, rotating the OpenAI key, troubleshooting.
- **Q-5 decision with you:** host the full stack or not. If yes: choose a provider with a free/cheap tier for API + managed Postgres, `DEMO_RESET_ENABLED`, daily narration cap, nightly reset; record D-047. If no: record D-047 "static demo only; full version local".

**Done when:** public static URL works on desktop + mobile; runbook followed from a clean clone succeeds (you or a fresh Codex task verify); D-047 recorded.

**Out of scope:** custom domains, monitoring stacks.

**Starter prompt:**
```
$whynot-session
Session 7.2 — Deploy, environments, runbook.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-7-ship.md (section "Session 7.2") fully before writing code.
Also read: docs/01-architecture.md (Runtime modes), docs/08-decisions.md.
Use skills: whynot-session (and a deploy skill only after I confirm hosting).
Work on branch session/7.2-deploy. Ask me the Q-5 hosting question before doing any hosting work.
Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 7.3 — Portfolio package → Release

**Goal:** someone who lands on the repo understands, believes and remembers it within two minutes.

**Depends on:** 7.2
**Read first:** `docs/00-product-brief.md`, `docs/03-engine-algorithms.md`, `docs/06-testing-and-accuracy.md` (What "accurate" means).
**Skills:** `whynot-session`, `screenshot`, `playwright`, `ui-ux-pro-max`.

**Build:**
- `README.md` rewrite: one-line pitch, live demo link, GIF of the H1 flow (≤ 15 s), 3 screenshots (patient panel, staff insights, inspector), "How it works" (diagram + 5 bullets), what's proven (properties) vs judged (helpfulness), tech stack, run locally, honest limitations, license.
- `docs/writeup.md`: technical article "Explaining a 'no': minimal explanations for booking rules" — the hitting-set idea, weights, alternatives cost model, privacy projection, verified AI wording, testing approach, what I'd do next.
- `docs/video-script.md`: 2-minute demo script (shot list + narration).
- Architecture diagram as Mermaid in README (and exported PNG in `docs/design/`).
- Final helpfulness review `docs/reviews/helpfulness-<date>.md` (you score H1–H8).
- Tag `v1.0.0` after merge; GitHub release notes.

**Done when:** release checklist in `docs/07-roadmap.md` fully ticked; `v1.0.0` tagged.

**Out of scope:** new features — list ideas in `docs/next.md` instead.

**Starter prompt:**
```
$whynot-session
Session 7.3 — Portfolio package (Release).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-7-ship.md (section "Session 7.3") fully before writing code.
Also read: docs/00-product-brief.md, docs/03-engine-algorithms.md, docs/06-testing-and-accuracy.md.
Use skills: whynot-session, screenshot, playwright, ui-ux-pro-max.
Work on branch session/7.3-portfolio. Implement exactly the "Build" list; nothing from "Out of scope".
Write in a clear, first-person, honest voice; no exaggerated claims beyond what tests prove.
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```
