---
name: whynot-session
description: Use at the start and end of EVERY WhyNot build session (any prompt mentioning "Session <id>" or a session card in docs/sessions). Defines how to read the spec, plan, implement, verify, hand off and open the PR. Do not use for review-only tasks (use whynot-review).
---

# WhyNot session protocol

## 1. Orient (before writing any code)

1. Read `AGENTS.md` fully.
2. Read `docs/PROGRESS.md`: confirm every session your card "Depends on" is ticked. If not, stop and report.
3. Read your session card in `docs/sessions/phase-*.md` completely, plus every doc in its "Read first" list.
4. Read the hand-off notes of the previous 2 sessions in `docs/PROGRESS.md`.
5. Read `docs/QUESTIONS.md`; if an open question affects your card, stop and report.
6. `git checkout main && git pull`, then `git checkout -b session/<id>-<slug>`.
7. **Skills check:** compare the skills your session card lists under "Skills" with the skills available to you. If any listed skill is missing (e.g. `playwright`, `openai-docs`, `ui-ux-pro-max`), STOP before coding and tell the human exactly which are missing and the install command from `docs/09-codex-workflow.md` (`$skill-installer <name>`, then restart Codex). Do not continue without them. Exception: Session 4.1 installs `ui-ux-pro-max` itself.
8. Run `pnpm verify` once on the untouched branch. If it is already red, stop and report — do not fix unrelated breakage silently.

## 2. Plan

Write a short plan (in your reply, not a file): files you will create/modify (must match the card), the order, and which tests prove each "Done when" item. If your plan needs a file the card does not mention, justify it in one line; if it needs an architectural change, stop and write in `docs/QUESTIONS.md`.

## 3. Implement in small, verified steps

- Contracts/types first, then implementation, then tests — or tests first for engine logic with clear expectations.
- After each meaningful step: `pnpm typecheck` and the relevant `pnpm test -- <path>`.
- Load the area skill: `whynot-engine`, `whynot-api`, `whynot-ui`, `whynot-narration`, and `whynot-testing` for tests.
- Never touch files belonging to later sessions. Never "improve" unrelated code.

## 4. Verify

- `pnpm verify` must be green.
- Walk every "Done when" item and confirm with evidence (test name, command output, screenshot for UI).
- Self-check against the `whynot-review` checklist quickly (scope, tests not weakened, invariants).

## 5. Hand off

1. Tick the session in `docs/PROGRESS.md` and add a hand-off note (2–4 lines): what was built, anything surprising, decisions made within the card's latitude, what the next session should know.
2. If the card says to update a doc (e.g. AGENTS.md commands), do it.
3. Commit (imperative, scoped messages; no AI attribution or Co-authored-by trailers), push, open a PR (use `$yeet` if installed). PR body: session ID, summary, verification output summary, open questions.

## Stop conditions (stop and report instead of continuing)

- Spec ambiguity or contradiction → `docs/QUESTIONS.md`.
- The same failure persists after 2 genuine fix attempts → write the failing case + hypothesis in `docs/QUESTIONS.md`.
- A golden expectation seems wrong → do not edit it; write in `docs/QUESTIONS.md`.
- A dependency session's output is missing or broken.
