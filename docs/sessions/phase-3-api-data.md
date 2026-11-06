# Phase 3 — API & data

Exit gate (checked in Session 3.4): API integration suite green on PGlite (and `@pg` tests green in CI); concurrent-booking test passes; H1–H8 reproduce through `POST /api/explain` on the seeded DB with the same outcomes as the engine goldens.

---

## Session 3.1 — DB schema, migrations, PGlite test harness, Docker Compose

**Goal:** the database exists with migrations, can be spun up in-process for tests, and in Docker for local dev.

**Depends on:** 2.5
**Read first:** `docs/05-api-and-data.md` (Database schema, Test database), `docs/08-decisions.md` (D-031, D-032, D-034, Q-2).
**Skills:** `whynot-session`, `whynot-api`, `whynot-testing`.

**Build:**
- `apps/api/src/db/schema.ts` (Drizzle) — all tables, indexes, the partial unique index for active policy, exclusion constraints (via custom SQL migration if Drizzle can't express them).
- `apps/api/drizzle.config.ts`, generated migrations in `apps/api/src/db/migrations/`, scripts `db:generate`, `db:migrate`.
- `apps/api/src/db/client.ts`: `createDb({ kind: 'pg', url } | { kind: 'pglite' })`.
- `apps/api/test/helpers/testDb.ts`: fresh migrated PGlite DB per test file.
- `docker-compose.yml`: Postgres 16 with volume and healthcheck; `.env.example` updated (`DATABASE_URL`).
- **Resolve Q-2:** test whether PGlite supports `btree_gist` exclusion constraints. If not, the constraint test is tagged `@pg`; add a CI job with a `postgres:16` service that runs `pnpm test -- --grep @pg`. Record the outcome as D-044.

**Tests:** migration applies cleanly on PGlite; round-trip insert/select per table; active-policy uniqueness; overlapping-booking rejection (PGlite or `@pg`).

**Done when:**
- [ ] `pnpm test` runs DB tests with no Docker.
- [ ] `docker compose up -d && pnpm db:migrate` works locally (documented in hand-off).
- [ ] D-044 recorded; CI green (including `@pg` job if created).

**Out of scope:** routes, seed data.

**Starter prompt:**
```
$whynot-session
Session 3.1 — DB schema, migrations, PGlite test harness, Docker Compose.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-3-api-data.md (section "Session 3.1") fully before writing code.
Also read: docs/05-api-and-data.md, docs/08-decisions.md.
Use skills: whynot-session, whynot-api, whynot-testing.
Work on branch session/3.1-db. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 3.2 — Seed script, demo clock, historical evaluations

**Goal:** one command fills the DB with Riverside Family Clinic exactly as the fixtures define it, plus two weeks of synthetic history for the dashboard.

**Depends on:** 3.1
**Read first:** `docs/05-api-and-data.md` (Demo clock, Insights queries), `docs/02-domain-and-rules.md` (Riverside clinic).
**Skills:** `whynot-session`, `whynot-api`, `whynot-testing`.

**Build:**
- `apps/api/src/seed/index.ts` (`pnpm db:seed`): truncates + inserts from `@whynot/fixtures` clinic builder (single source of truth — no duplicated data), active policy v1.
- Historical evaluations: generate ~1,500 requests (single-slot and some range requests) over the 14 days before `DEMO_DATE` from a seeded generator, run the engine on each (against the snapshot as of that time), insert into `evaluations`. Deterministic.
- `apps/api/src/clock.ts`: `getNow()` per doc 05 (`DEMO_NOW` frozen, else `DEMO_DATE` + real time-of-day); env documented in `.env.example`.
- `apps/api/src/snapshotLoader.ts`: `loadSnapshot(db, { from, to })` → contract-validated `ClinicSnapshot` (historical mode: `asOf` filters bookings by `created_at`/`cancelled_at`).

**Tests:** seed is idempotent and deterministic (two runs ⇒ identical row hashes); `loadSnapshot` on the seeded DB deep-equals the fixtures snapshot for the same window; historical `asOf` filter; clock modes.

**Done when:**
- [ ] `pnpm db:seed` completes in < 30 s locally.
- [ ] Snapshot-equality test passes (proves DB ↔ fixtures fidelity).
- [ ] `pnpm verify` green.

**Out of scope:** HTTP routes.

**Starter prompt:**
```
$whynot-session
Session 3.2 — Seed script, demo clock, historical evaluations.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-3-api-data.md (section "Session 3.2") fully before writing code.
Also read: docs/05-api-and-data.md, docs/02-domain-and-rules.md (Riverside clinic).
Use skills: whynot-session, whynot-api, whynot-testing.
Work on branch session/3.2-seed. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 3.3 — API skeleton, `/clinic`, `/availability`, `/explain`, `/explain-range`

**Goal:** the read side of the API works end to end with validation, audience projection and evaluation logging.

**Depends on:** 3.2
**Read first:** `docs/05-api-and-data.md` (Conventions, Endpoints), `docs/04-explanation-and-ai.md` (Audience projection).
**Skills:** `whynot-session`, `whynot-api`, `whynot-testing`, `security-threat-model` (draft only).

**Build:**
- `packages/contracts/src/api.ts`: request/response schemas for `health`, `clinic`, `availability`, `explain`, `explain-range`, and the error envelope.
- `apps/api/src/app.ts`: `buildApp({ db, clock, config })` — Fastify with zod type provider, error handler, request logging (no PII), body limit, CORS, rate-limit plugin.
- Routes: `GET /api/health`, `GET /api/clinic`, `GET /api/availability`, `POST /api/explain` (stores staff-level evaluation, returns projected + `evaluationId`), `POST /api/explain-range` (projected `RangeExplanation`; stores one evaluation row per range with `outcome = fully_blocked|has_availability` and the common-cause/segment rule IDs in `primary_rule_ids`).
- `apps/api/src/demoRole.ts`: header parsing + guards.
- `docs/security/threat-model.md`: first draft using `security-threat-model` (assets, trust boundaries, abuse cases) — to be finalised in 7.1.

**Tests:** each route (success, validation error, unknown id); patient vs staff projection on H1; availability grid matches `isBookable` for every slot of a day; **H1–H8 through `/api/explain` and H9–H11 through `/api/explain-range` match engine goldens**.

**Done when:**
- [ ] All listed tests pass; response bodies parse against contract schemas.
- [ ] Patient responses for all goldens pass the P10 forbidden-string scan.
- [ ] `pnpm verify` green.

**Out of scope:** bookings, admin routes, narration.

**Starter prompt:**
```
$whynot-session
Session 3.3 — API skeleton, /clinic, /availability, /explain, /explain-range.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-3-api-data.md (section "Session 3.3") fully before writing code.
Also read: docs/05-api-and-data.md, docs/04-explanation-and-ai.md (Audience projection).
Use skills: whynot-session, whynot-api, whynot-testing, security-threat-model.
Work on branch session/3.3-api-read. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

---

## Session 3.4 — Bookings, cancel, policies, demo reset → Phase 3 gate

**Goal:** the write side is race-safe and auditable; policy versions are immutable; the demo can reset itself.

**Depends on:** 3.3
**Read first:** `docs/05-api-and-data.md` (Booking creation and race safety, Endpoints), `docs/08-decisions.md` (D-034).
**Skills:** `whynot-session`, `whynot-api`, `whynot-testing`, `security-best-practices`.

**Build:**
- Contracts for bookings, policies, reset.
- `POST /api/bookings` (serializable transaction flow + retry + 409 with projected evaluation), `DELETE /api/bookings/:id` (patient: own only; staff: any; soft cancel).
- `GET/POST /api/admin/policies` (new version + activate in one transaction; staff only).
- `POST /api/demo/reset` (env-gated, rate-limited; re-runs seed).
- `GET /api/admin/evaluations/:id/trace` (re-evaluate stored request against historical snapshot with `trace: true`).

**Tests:** booking success → slot now blocked with `practitioner-busy`; **concurrency:** 10 parallel requests for the last free slot ⇒ exactly 1 created, 9 × 409 (on real Postgres via `@pg` if PGlite can't run concurrent transactions — note in hand-off); cancel permissions; new policy version changes `/explain` result (e.g. min-notice 120 → 30 makes H3 bookable); reset restores counts; staff-only routes 403 for patients.

**Done when (Phase 3 gate):**
- [ ] All tests pass (incl. `@pg` in CI).
- [ ] H1–H8 still match through the API after a reset.
- [ ] `security-best-practices` pass on `apps/api` with findings fixed or listed in `docs/QUESTIONS.md`.
- [ ] Gate result written in `docs/PROGRESS.md`.

**Out of scope:** insights, simulator, narration.

**Starter prompt:**
```
$whynot-session
Session 3.4 — Bookings, cancel, policies, demo reset (Phase 3 gate).
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/phase-3-api-data.md (section "Session 3.4") fully before writing code.
Also read: docs/05-api-and-data.md, docs/08-decisions.md (D-034).
Use skills: whynot-session, whynot-api, whynot-testing, security-best-practices.
Work on branch session/3.4-api-write. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md (including the Phase 3 gate result) and opening a PR with $yeet.
```
