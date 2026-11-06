# Phase 5 — AI narration (OpenAI)

Exit gate (checked in Session 5.3): verifier adversarial suite green; narration eval ≥ 90% first-pass verification and **0 privacy failures** across all goldens × both audiences; model choice recorded (Q-4 → D-046); static demo shows verified AI narrations for H1–H8.

Before writing OpenAI code, use `openai-docs` to confirm current Responses API and structured-output syntax. Do not rely on memory.

---

## Session 5.1 — Narrator interface, OpenAI narrator, cache, budget

**Goal:** the server can ask OpenAI to reword a projected explanation into structured, cited sentences — cheaply, cached and bounded.

**Depends on:** 4.5
**Read first:** `docs/04-explanation-and-ai.md` (AI narration — all), `.agents/skills/whynot-narration/SKILL.md`.
**Skills:** `whynot-session`, `whynot-narration`, `openai-docs`, `security-best-practices`.

**Build:**
- `packages/narrator/src/prompt.ts`: system prompt + 3 few-shots (from H1, H3, H6 template outputs), `PROMPT_VERSION = '1'`, JSON schema for output.
- `packages/narrator/src/openai.ts`: `OpenAINarrator({ client, model, timeoutMs, maxOutputTokens })` — Responses API with strict JSON schema; one retry on transport errors; returns raw structured output **unverified** (verification wired in 5.2). Uses the official `openai` npm package; config injected (no `process.env` in the package).
- `apps/api/src/config.ts`: `OPENAI_API_KEY`, `OPENAI_MODEL`, `NARRATION_ENABLED`, `NARRATION_DAILY_CAP`; `.env.example` updated.
- `apps/api` `narrations` cache (hash key per doc 04) and daily-cap counter.
- `POST /api/narrate` (contract in `api.ts`): for now returns template narration always, plus — when enabled and in staff role only — the raw AI output under a `debug` field (removed in 5.2). Rate limited.

**Tests:** OpenAI narrator with a fake client (recorded fixture responses): request shape (model, schema, max tokens, input contains only projected facts), timeout → error, transport retry once; cache hit avoids second call; cap reached → template.

**Done when:**
- [ ] Manual check with a real key (you run it): `/api/narrate` on H1 returns structured sentences in `debug`; cost per call noted in hand-off.
- [ ] No test needs a real key; `pnpm verify` green.
- [ ] Lint confirms `apps/web` can't import `openai.ts`.

**Out of scope:** verifier; UI toggle; eval runner.

**Starter prompt:**
```
$whynot-session
Session 5.1 — Narrator interface, OpenAI narrator, cache, budget.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-5-ai-narration.md (section "Session 5.1") fully before writing code.
Also read: docs/04-explanation-and-ai.md (AI narration).
Use skills: whynot-session, whynot-narration, openai-docs, security-best-practices.
Use openai-docs to confirm current Responses API structured-output syntax before coding.
Work on branch session/5.1-openai-narrator. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 5.2 — Verifier, adversarial tests, fallback wiring

**Goal:** no AI sentence reaches a user unless it is grounded in the engine's facts; otherwise templates are shown — automatically.

**Depends on:** 5.1
**Read first:** `docs/04-explanation-and-ai.md` (Verifier table), `.agents/skills/whynot-narration/SKILL.md`.
**Skills:** `whynot-session`, `whynot-narration`, `whynot-testing`, `whynot-ui`.

**Build:**
- `packages/narrator/src/verify.ts`: every check in doc 04's table → `VerificationReport { passed, failures: { check, sentenceIndex, detail }[] }`.
- `packages/narrator/src/narrate.ts`: `narrateVerified(input, { ai?, template })` → AI if verified, else template (with report).
- `/api/narrate` returns the verified narration for the caller's audience; `debug` field removed; verification reports stored.
- Web: wording toggle in the Why-not panel ("Plain" / "AI wording — verified"), label shows actual source; falls back silently with a small "(plain wording shown)" note when AI failed.

**Tests:** adversarial suite (≥ 25 fake outputs): invented time, invented weekday, wrong practitioner, other patient's name, staff word ("leave") for patient, missing citation, citing a non-existent fact, uncovered primary fact, promise phrase, too long, malformed JSON — each must fail the right check; valid paraphrases must pass (≥ 10). Integration: `/api/narrate` returns template when fake AI output fails.

**Done when:**
- [ ] Adversarial suite green; no check loosened to make valid paraphrases pass (justify any false-rejection in hand-off).
- [ ] UI toggle works in full mode (screenshot).
- [ ] `pnpm verify` green.

**Out of scope:** eval runner; static narrations.

**Starter prompt:**
```
$whynot-session
Session 5.2 — Verifier, adversarial tests, fallback wiring.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-5-ai-narration.md (section "Session 5.2") fully before writing code.
Also read: docs/04-explanation-and-ai.md (Verifier).
Use skills: whynot-session, whynot-narration, whynot-testing, whynot-ui.
Work on branch session/5.2-verifier. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 5.3 — Narration eval, model choice, static narrations → Phase 5 gate

**Goal:** measure AI wording quality honestly, pick the model on evidence, and ship verified narrations to the static demo.

**Depends on:** 5.2
**Read first:** `docs/06-testing-and-accuracy.md` (Narration eval), `docs/08-decisions.md` (D-020, Q-4).
**Skills:** `whynot-session`, `whynot-narration`, `openai-docs`.

**Build:**
- `tools/eval-narration/`: `pnpm eval:narration --model <id> [--runs 3]` over all goldens × audiences; report `reports/narration-<date>-<model>.md` (pass rate, failures by check, privacy failures, side-by-side texts, tokens, estimated cost).
- Run it (you provide the key) for 2–3 candidate small/cheap OpenAI models; pick the cheapest meeting the gate; record as D-046 (model, pass rate, cost per 1,000 narrations). Close Q-4.
- Iterate the prompt (bump `PROMPT_VERSION`) if pass rate < 90% — never loosen the verifier.
- Generate `apps/web/src/static/narrations.json` (verified only, keyed by golden ID + audience); static adapter uses it when the request matches a golden.

**Done when (Phase 5 gate):**
- [ ] Report committed; first-pass ≥ 90%; privacy failures = 0.
- [ ] D-046 recorded; `.env.example` default model updated.
- [ ] Static demo shows "AI wording — verified" for H1–H11.
- [ ] Gate result written in `docs/PROGRESS.md`.

**Out of scope:** new verifier checks unless a real failure mode is found (then add with a test).

**Starter prompt:**
```
$whynot-session
Session 5.3 — Narration eval, model choice, static narrations (Phase 5 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-5-ai-narration.md (section "Session 5.3") fully before writing code.
Also read: docs/06-testing-and-accuracy.md (Narration eval), docs/08-decisions.md.
Use skills: whynot-session, whynot-narration, openai-docs.
Work on branch session/5.3-narration-eval. Implement exactly the "Build" list; nothing from "Out of scope".
Ask me to run the eval with my API key when the runner is ready; never commit keys.
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 5 gate result) and opening a PR with $yeet.
```
