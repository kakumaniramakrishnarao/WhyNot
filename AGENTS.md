# AGENTS.md — WhyNot

Standing instructions for every coding agent working in this repository. Read fully before any change.

## What this project is

WhyNot is an explainable booking engine for a fictional clinic ("Riverside Family Clinic"). When a booking request fails, it returns the **minimal set of rules that block it**, **verified bookable alternatives**, and for staff a **minimal policy fix**. AI (OpenAI) only rewords results; a verifier checks every sentence.

The full plan is in `docs/` — `docs/00`–`09` are the specification. **The docs are the source of truth. Code follows docs.** If they disagree, stop and write the conflict in `docs/QUESTIONS.md`.

## Every session

1. Use `$whynot-session`. Read `docs/PROGRESS.md` and your session card in `docs/sessions/` completely.
2. Implement **only** the card's "Build" list. Never implement items from "Out of scope" or from later sessions "while you're there".
3. Run `pnpm verify` and make it green. Never finish with red checks.
4. Update `docs/PROGRESS.md` (tick + 2–4 line hand-off note: what was done, anything surprising, what the next session should know).
5. Commit on branch `session/<id>-<slug>`, push, open a PR.

If the spec is ambiguous, contradictory, or seems wrong: **stop, write the question in `docs/QUESTIONS.md`, and do only the unambiguous part.** Never guess about architecture, contracts, rule semantics, or test expectations.

## Commands

(Filled in precisely by Session 0.1. Target set:)

| Command | Does |
| --- | --- |
| `pnpm install` | Install workspace |
| `pnpm verify` | format check → lint → typecheck → test → build. **The gate.** |
| `pnpm test` | All Vitest suites (unit, property, golden, integration on PGlite) |
| `pnpm test:e2e` | Playwright (needs built web app) |
| `pnpm bench` | Engine benchmarks |
| `pnpm dev` | API + web in watch mode (needs Postgres from `docker compose up -d`) |
| `pnpm db:migrate` / `pnpm db:seed` | Apply migrations / seed Riverside clinic |
| `pnpm eval:narration` | Narration eval (needs `OPENAI_API_KEY`) |

## Architecture invariants (never violate)

1. **The engine decides; nothing else does.** Feasibility, reasons, alternatives and admin fixes come only from `packages/engine`. API, web and AI must never compute, filter, reorder or override them (presentation formatting is fine).
2. **`packages/engine` is pure.** No I/O, no `Date.now()`, no `Math.random()`, no `process`, no Node built-ins, no network, no DB. Allowed imports: `@whynot/contracts`, `luxon` (only inside `src/time.ts`). `now` is always a parameter.
3. **Deterministic.** Sort inputs canonically; break ties by stable IDs. Same input ⇒ deep-equal output.
4. **Time:** inside the engine, instants are integer **epoch minutes UTC**. Wall-clock conversion happens only in `engine/src/time.ts`. On the wire: ISO-8601 with offset.
5. **Contracts first.** Shared types are zod schemas in `packages/contracts`. Never redefine a contract type elsewhere; infer with `z.infer`.
6. **Privacy:** anything shown to a patient goes through `projectForAudience(…, 'patient')`. Never send staff-only facts, other patients' data, booking IDs, or `reason` fields to patient surfaces.
7. **Templates first.** Every rule has patient and staff templates. AI narration is optional and must pass `verify()`; otherwise use templates.
8. **No secrets in the browser.** `OPENAI_API_KEY` exists only in `apps/api`. `apps/web` never imports `narrator/src/openai.ts`.
9. **Dependency direction:** `contracts ← engine ← narrator ← apps`. `fixtures` is dev/test only. Enforced by ESLint; never disable those rules.
10. **No new infrastructure** (queues, caches, microservices, vector DBs, agent frameworks, state libraries beyond TanStack Query) without a decision-log entry approved by the human.

## Testing rules

- New behaviour ⇒ new tests in the same PR. Engine changes ⇒ unit + property tests (see `$whynot-testing`).
- **Golden scenarios** (`packages/fixtures/scenarios/*.json`) encode human expectations. **Never edit an expectation to make a test pass.** Only change one together with an entry in `docs/08-decisions.md` explaining why the expectation was wrong, and say so in the PR description.
- Never use `.skip`, `.only`, `it.todo`, `@ts-ignore`, `@ts-expect-error` (except in type tests), `eslint-disable`, or lowered property-run counts to get green.
- Property tests print their seed; on failure, add the shrunk counterexample as a regular unit test.
- Do not mock the engine in API or web tests; use real fixtures. Mock only OpenAI (with recorded/adversarial responses).

## Code conventions

- TypeScript `strict`, no `any` (use `unknown` + zod). Named exports only. ESM.
- Files: `camelCase.ts`; React components `PascalCase.tsx`. One rule family per file in `engine/src/rules/`.
- Pure functions over classes in the engine. Fastify plugins per route group in the API.
- Errors: engine never throws for business outcomes (returns `invalid_request`/`blocked`); throws only on bugs/invariant violations.
- Comments explain *why*, not *what*. Public engine functions get TSDoc.
- Keep functions small; no file over ~300 lines without a reason.
- Formatting: Prettier defaults (+ `printWidth: 100`). Don't hand-format.

## Git & PR conventions

- Branch: `session/<id>-<slug>` (e.g. `session/1.4-resource-rules`). Never commit directly to `main`.
- Commits: imperative, scoped: `engine: add practitioner buffer rule`. Small logical commits are fine; PRs are squash-merged.
- **Do not add `Co-authored-by` trailers or any AI-tool attribution to commits, PRs or code.** The human is the sole author.
- PR description: session ID, what was built, how it was verified (paste `pnpm verify` summary), anything left in `docs/QUESTIONS.md`.
- Don't modify `docs/00`–`09` unless the session card says so or a decision-log entry requires it.

## Skills

Project skills live in `.agents/skills/`. Use them:

- `$whynot-session` — every session (always).
- `$whynot-engine` — `packages/engine`, `packages/contracts`.
- `$whynot-testing` — any test work.
- `$whynot-api` — `apps/api`.
- `$whynot-ui` — `apps/web` (together with `ui-ux-pro-max`).
- `$whynot-narration` — `packages/narrator`, `tools/eval-narration`.
- `$whynot-review` — reviewing a finished session in a fresh task.

**Before starting any session, check that every skill the session card lists is available. If one is missing, stop and tell the human which one and how to install it (`docs/09-codex-workflow.md`).**

External skills (install per `docs/09-codex-workflow.md`): `ui-ux-pro-max`, `playwright`, `playwright-interactive`, `screenshot`, `security-best-practices`, `security-threat-model`, `openai-docs`, `gh-fix-ci`, `gh-address-comments`, `yeet`.

## Things that are deliberately out of scope

Real auth, real patient data, payments, notifications, calendar sync, multi-clinic, recurring bookings, third-party scheduler integration, AI making decisions or bookings. Don't add them.
