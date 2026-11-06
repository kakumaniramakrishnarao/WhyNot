# 05 — API & data

## Conventions

- JSON over HTTP, base path `/api`. Every request and response body has a zod schema in `packages/contracts/src/api.ts`; Fastify validates both directions (response validation on in dev/test).
- Instants on the wire: ISO-8601 strings with offset (e.g. `2026-11-10T14:30:00+00:00`). Converted to epoch minutes at the boundary.
- Errors: `{ error: { code: string, message: string, details?: unknown } }` with proper status codes (400 validation, 404 unknown id, 409 booking conflict, 429 rate limit, 500 bug).
- **Audience** is chosen by a demo role header `X-Demo-Role: patient | staff` (+ `X-Demo-Patient: <id>` for patients). This is a demo role switcher, **not** authentication; it is labelled as such in the UI and docs. Server applies `projectForAudience` based on it.

## Endpoints

| Method & path | Purpose | Notes |
| --- | --- | --- |
| `GET /api/health` | Liveness + engine version + DB ok | |
| `GET /api/clinic` | Public clinic info: services, practitioners (public fields), opening hours, demo clock | |
| `GET /api/availability?serviceId&date&practitionerId?` | Slot grid for a day: `[{ start, bookable }]` | Uses `isBookable` per grid slot; cached per (policyVersion, date, service, practitioner, bookingsVersion) |
| `POST /api/explain` | Body `{ serviceId, start, practitionerId? }` (+ patient from header) → projected `Evaluation` + `evaluationId` | Stores staff-level evaluation in `evaluations` |
| `POST /api/narrate` | Body `{ evaluationId }` → `Narration` for caller's audience | Template if AI disabled/over budget/verification fails |
| `POST /api/bookings` | Body `{ serviceId, start, practitionerId? }` → booking or 409 with fresh explanation | Re-evaluates inside a transaction (see below) |
| `DELETE /api/bookings/:id` | Cancel own booking (patient) or any (staff) | Demo convenience |
| `GET /api/admin/insights?from&to&serviceId?` | Aggregates over `evaluations` | staff only |
| `POST /api/admin/simulate` | Body `{ from, to, policyChanges }` → simulation result | staff only; read-only |
| `GET /api/admin/policies` / `POST /api/admin/policies` | List versions / create a new version and activate it | staff only; never mutates old versions |
| `GET /api/admin/evaluations/:id/trace` | Re-run with `trace: true` on the stored snapshot inputs | staff only; inspector |
| `POST /api/demo/reset` | Restore seeded state | rate limited; enabled by env `DEMO_RESET_ENABLED` |

## Booking creation and race safety

`POST /api/bookings`:

1. Open a transaction at `SERIALIZABLE` isolation.
2. Load the snapshot **inside** the transaction; run `evaluate`.
3. If bookable → insert booking with the engine's assignment; commit.
4. If blocked → rollback; respond 409 with the projected evaluation (so the UI shows "just taken — here's why and here are alternatives").
5. On serialization failure → retry up to 3 times, then 409.

Backstop: Postgres exclusion constraints (`btree_gist`) on `bookings` prevent overlapping confirmed bookings for the same practitioner/room/equipment, even if application logic had a bug. (Buffers are not DB-enforced; the engine owns them.)

## Database schema (Drizzle, PostgreSQL 16)

```
clinic            (id pk, name, timezone, slot_grid_min, opening_hours jsonb, closed_dates date[])
practitioners     (id pk, display_name, role, skills text[], working_hours jsonb, breaks jsonb, daily_cap int null)
time_off          (id pk, practitioner_id fk, starts_at timestamptz, ends_at timestamptz, reason text)
rooms             (id pk, display_name, capabilities text[])
equipment         (id pk, display_name, type text)
maintenance       (id pk, resource_type text check in ('room','equipment'), resource_id text, starts_at, ends_at)
services          (id pk, display_name, duration_min, required_skill, required_room_capability null,
                   required_equipment_type null, buffer_after_min, room_turnover_min, eligibility)
patients          (id pk, display_name, is_returning bool, referrals text[])
policy_versions   (id pk, created_at, params jsonb, is_active bool, note text)   -- exactly one active (partial unique index)
bookings          (id pk, patient_id fk, service_id fk, practitioner_id fk, room_id fk, equipment_id fk null,
                   starts_at, ends_at, status, created_at, cancelled_at null,
                   EXCLUDE USING gist (practitioner_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE status='confirmed',
                   ... same for room_id, equipment_id)
evaluations       (id pk, created_at, patient_id, request jsonb, outcome text, primary_rule_ids text[],
                   evaluation jsonb /* staff-level */, policy_version_id fk, engine_version text, duration_ms int)
narrations        (cache_key pk, created_at, audience, model, prompt_version, narration jsonb, verification jsonb)
```

Indexes: `bookings(practitioner_id, starts_at)`, `bookings(room_id, starts_at)`, `bookings(patient_id, starts_at)`, `evaluations(created_at)`, GIN on `evaluations(primary_rule_ids)`.

Snapshot loader (`apps/api/src/snapshotLoader.ts`) reads only the window needed: request day − 1 to request day + `searchDays` + 1, plus everything static. One function, unit-tested against fixtures.

## Demo clock

- The demo lives in a fixed **demo week**. `DEMO_DATE` (env, default a fixed Monday) anchors the seed data.
- The server's `now` for engine calls = `DEMO_DATE + (real time-of-day offset)` by default, or fully frozen (`DEMO_NOW` env) for tests/e2e. So "in 1 hour" (H3) keeps working forever.
- The web app shows the demo date/time in the header ("Demo clinic · Tuesday 10 Nov 2026, 09:40").

## Test database: PGlite

All API integration tests run against **PGlite** (Postgres compiled to WASM, in-process, via Drizzle's PGlite driver) — no Docker needed, so tests run in any sandbox including Codex cloud. Docker Compose Postgres is for local dev/demo. If a needed Postgres feature (e.g. `btree_gist` exclusion constraints) is unavailable in PGlite, that specific test is tagged `@pg` and runs in CI against a Postgres service container; the decision is recorded in the decision log during Session 3.1.

## Insights queries

- **Top blocking rules:** count of `blocked` evaluations by each rule ID in `primary_rule_ids`, over a window, optionally per service/practitioner/weekday.
- **Blocked share:** blocked / total evaluations per day.
- **Near misses:** blocked evaluations whose `adminFix` exists, grouped by the setting it would change ("the 15-min procedure buffer blocked 23 requests that had an otherwise free slot").

Insights are computed with SQL over `evaluations`; the seed script generates ~2 weeks of synthetic historical evaluations (by running the engine over generated requests) so the dashboard is populated on first run.

## Policy simulator

`POST /api/admin/simulate { from, to, policyChanges }`:

1. Load past evaluations in the window (requests only).
2. For each request, re-run `evaluate` against the **historical** snapshot with the modified policy. Historical bookings = `created_at ≤ evaluation time AND (cancelled_at IS NULL OR cancelled_at > evaluation time)`; cancellation is a soft status change, never a delete, so history stays reconstructable.
3. Return `{ total, previouslyBlocked, nowBookable, nowBlocked, byRule }` plus examples.
4. Always labelled **"estimate — each request replayed independently; ignores knock-on effects"**.

Bounded: max 2,000 requests per simulation; runs synchronously (target < 3 s on demo data).
