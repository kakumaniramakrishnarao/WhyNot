# Phase 0 — Foundation

Exit gate: `pnpm verify` green locally and in CI; import-boundary and engine-purity lint rules demonstrably fail on a violating sample; project skills visible in Codex.

---

## Session 0.1 — Monorepo scaffold, tooling, `pnpm verify`, CI

**Goal:** an empty but fully wired TypeScript monorepo where `pnpm verify` runs format, lint, typecheck, test and build across all packages, locally and in GitHub Actions.

**Depends on:** —
**Read first:** `AGENTS.md`, `docs/01-architecture.md` (Repository layout, Tech stack), `docs/06-testing-and-accuracy.md` (The single gate), `docs/08-decisions.md` (D-030, D-031, D-042, Q-1).
**Skills:** `whynot-session`, `whynot-testing`.

**Build:**
- Root: `package.json` (private, `"type": "module"`, `packageManager` pinned pnpm, `engines.node` ≥ 22), `pnpm-workspace.yaml` (`packages/*`, `apps/*`, `tools/*`), `tsconfig.base.json` (strict, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, ESM, `moduleResolution: bundler`), `.nvmrc` (22), `.gitignore`, `.editorconfig`, `.prettierrc` (`printWidth: 100`), `eslint.config.js` (flat config, typescript-eslint strict), `vitest.workspace.ts`, `LICENSE` (MIT, copyright holder: the repository owner), `.env.example` (empty sections with comments).
- Packages (empty skeletons with `package.json`, `tsconfig.json`, `src/index.ts` exporting a `VERSION` const, one smoke test each): `packages/contracts` (`@whynot/contracts`, depends on zod), `packages/engine` (`@whynot/engine`), `packages/narrator` (`@whynot/narrator`), `packages/fixtures` (`@whynot/fixtures`).
- Apps skeletons: `apps/api` (`@whynot/api`, Fastify hello `/api/health` + test via `inject`), `apps/web` (`@whynot/web`, Vite + React + TS, renders "WhyNot", builds).
- Root scripts: `format`, `format:check`, `lint`, `typecheck`, `test`, `build`, `verify` (runs the five in order, fails fast), `test:e2e` (placeholder that exits 0 with a message until 4.5), `bench` (placeholder until 1.6).
- `.github/workflows/ci.yml`: on PR and push to main; Node from `.nvmrc`; `corepack enable`; pnpm cache; `pnpm install --frozen-lockfile`; `pnpm verify` with `FC_NUM_RUNS=5000`.
- Update `AGENTS.md` → "Commands" table with the exact working commands (remove the "Filled in by Session 0.1" line). Record pinned versions (Node, pnpm, TypeScript, zod, Fastify, Vite, React, Vitest) as decision **D-043** in `docs/08-decisions.md` and close Q-1.

**Tests:** one smoke test per package/app (imports `VERSION`; API health returns 200 `{ status: 'ok' }`).

**Done when:**
- [ ] `pnpm install && pnpm verify` green from a clean clone.
- [ ] CI workflow runs green on the PR.
- [ ] `pnpm -r build` produces outputs for all packages and the web app.
- [ ] AGENTS.md command table matches reality; D-043 recorded.

**Out of scope:** any domain code, DB, Tailwind/design, Playwright, Docker Compose.

**Starter prompt:**
```
$whynot-session
Session 0.1 — Monorepo scaffold, tooling, pnpm verify, CI.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-0-foundation.md (section "Session 0.1") fully before writing code.
Also read: docs/01-architecture.md (Repository layout, Tech stack), docs/06-testing-and-accuracy.md (The single gate), docs/08-decisions.md.
Use skills: whynot-session, whynot-testing.
Work on branch session/0.1-scaffold. Implement exactly the "Build" list; nothing from "Out of scope".
Pin current stable versions and record them as D-043.
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 0.2 — Guard rails

**Goal:** make the architecture rules mechanically enforced so later sessions (and agents) cannot quietly break them.

**Depends on:** 0.1
**Read first:** `AGENTS.md` (Architecture invariants 2, 8, 9), `docs/01-architecture.md` (Dependency rules), `docs/09-codex-workflow.md`.
**Skills:** `whynot-session`.

**Build:**
- ESLint `no-restricted-imports` / `import/no-restricted-paths` per package implementing the dependency rules: engine may import only `@whynot/contracts` and `luxon` (luxon only from `src/time.ts`); narrator may not import `apps/*` or DB libs; web may not import `@whynot/narrator/openai` or any `apps/api` code; nothing imports `@whynot/fixtures` from non-test production code except `apps/api/src/seed` and `apps/web/src/static`.
- ESLint `no-restricted-globals` / `no-restricted-syntax` for `packages/engine/src/**`: ban `Date`, `Math.random`, `process`, `console`, `setTimeout`, `setInterval`, `fetch`, `require`.
- ESLint ban on `.only`/`.skip` in tests (`vitest` plugin rules), and `@typescript-eslint/no-explicit-any` as error.
- `tools/lint-fixtures/` with intentionally violating sample files + a test that runs ESLint programmatically on them and asserts the expected errors (proves the guard rails work). Exclude these samples from normal lint.
- `packages/engine/AGENTS.md`: short area rules (purity, determinism, epoch minutes) pointing to `$whynot-engine`.
- `.github/pull_request_template.md`: Session ID, What was built, Verification (`pnpm verify` summary), Done-when checklist copied from the card, Open questions, Screenshots (UI).
- `docs/adr-template.md` (one-paragraph decision entry format matching doc 08).

**Tests:** the guard-rail test in `tools/lint-fixtures`.

**Done when:**
- [ ] Violating samples produce the expected lint errors (test green); real code lints clean.
- [ ] `pnpm verify` green; CI green.
- [ ] Opening Codex in the repo lists the seven `whynot-*` skills (note in hand-off).

**Out of scope:** domain code; any change to docs 00–09.

**Starter prompt:**
```
$whynot-session
Session 0.2 — Guard rails.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-0-foundation.md (section "Session 0.2") fully before writing code.
Also read: docs/01-architecture.md (Dependency rules), docs/09-codex-workflow.md.
Use skills: whynot-session.
Work on branch session/0.2-guard-rails. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```
