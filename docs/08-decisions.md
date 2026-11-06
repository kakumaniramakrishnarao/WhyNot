# 08 — Decision log

Format: **ID · Decision** — *Why* — *Rejected alternatives* — *Revisit if*. New decisions are appended; superseded ones are marked, never deleted.

## Product

**D-001 · Build WhyNot (explainable booking engine) instead of the VouchDesk support-investigation agent.**
*Why:* VouchDesk's category is crowded (Pluno, Pylon, Intercom Fin) and its plan needed months of third-party integration work; WhyNot has an original combination, a provable core, and a 60-second demo.
*Rejected:* VouchDesk; generic RAG/chat apps; agent replay debugger (good, but less demo-friendly).
*Revisit if:* never for v1.

**D-002 · Portfolio-quality product first, not a startup.** Commercial adoption is unlikely (platforms own the rules). *Revisit if* real demand appears after launch.

**D-003 · Demo business: fictional Riverside Family Clinic.** Clinics have the richest interacting rules (skills, rooms, equipment, buffers, referrals). *Rejected:* gym, coworking.

**D-004 · Single clinic, single timezone, single-appointment requests in v1.** Keeps the algorithm exact and fast. Extension notes in doc 03.

**D-006 · Position WhyNot on the three moments where hiding slots fails** (patient stuck on an empty day/practitioner, front-desk calls, invisible lost bookings) — not on the ordinary "pick another open slot" case. Adds range explanations ("Why is this empty?", Session 2.5) as a first-class feature. *Why:* most apps hide unavailable slots; a single-slot explainer alone would solve a rare problem.

**D-005 · The demo creates bookings in its own DB; WhyNot integrates with no third-party scheduler in v1.**

## Engine

**D-010 · Pure TypeScript engine, no external solver.** Demo-sized candidate matrices (≤ ~100 candidates) are solved exactly with a hitting-set branch-and-bound. Enables in-browser static demo. *Rejected:* OR-Tools/Python service (second language, deployment cost), MiniZinc, clingo. *Revisit if* bench L dataset matters or multi-resource/series bookings are added.

**D-011 · Explanation = minimum-weight irredundant hitting set over the violation matrix.** Sufficient + minimal by construction; weights encode "simplest true thing". *Rejected:* first-failing-rule (misleading), all violations (noisy), LLM-chosen reasons (unverifiable).

**D-012 · Buffers/turnover are separate rules from busy.** Lets explanations distinguish "booked" from "policy buffer" and powers admin fixes.

**D-013 · Instants are integer epoch minutes (UTC) inside the engine; luxon isolated in `time.ts`.** *Rejected:* Date objects everywhere (TZ bugs), Temporal polyfill (heavier; revisit when Temporal is universal).

**D-014 · Alternatives ranked by an explicit, configurable cost model with category diversity.** "Nearest" is a preference; making it explicit makes it honest and tunable.

**D-015 · `now` is always injected; demo uses a demo clock.** Determinism and evergreen demo scenarios.

## AI

**D-020 · OpenAI API for optional narration (user choice: cost).** Behind a `Narrator` interface; model via `OPENAI_MODEL` env, chosen by the narration eval in Session 5.3. *Revisit if* cost/quality changes — swapping providers touches one file.

**D-021 · Templates are the default and the ground truth; AI output must pass a deterministic verifier or is replaced by templates.** The model never decides anything.

**D-022 · Static demo ships pre-verified narrations; no API key ever in the browser.**

## Platform

**D-030 · TypeScript monorepo with pnpm workspaces** (`packages/contracts`, `engine`, `narrator`, `fixtures`; `apps/api`, `apps/web`). *Rejected:* Nx/Turborepo (unneeded at this size), separate repos.

**D-031 · Fastify + Drizzle + PostgreSQL 16; React + Vite + TanStack Query + Tailwind; Recharts.**

**D-032 · API tests use PGlite (in-process Postgres), so no Docker is required in any sandbox.** Postgres-only features get `@pg`-tagged tests run in CI with a service container.

**D-033 · Demo role switcher instead of real authentication.** No real users or data exist. Clearly labelled. *Revisit if* hosted with real users (never planned).

**D-034 · Double safety for booking races:** serializable transaction re-evaluation + Postgres exclusion constraints.

**D-035 · Hosting decided in Session 7.2.** Static demo on GitHub Pages is fixed; full-stack hosting optional.

## Process

**D-040 · Built with Codex. Rules live in `AGENTS.md`; project skills live in `.agents/skills/`.** Curated OpenAI skills + UI UX Pro Max are installed per `docs/09-codex-workflow.md`.

**D-041 · One session = one verified, pushed increment; `pnpm verify` is the single gate; golden expectations change only with a decision-log entry.**

**D-042 · License: MIT for original code** (confirm in Session 0.1). No GPL dependencies in `packages/engine`.

## Open (to decide in the named session)

| ID | Question | Session |
| --- | --- | --- |
| Q-1 | Exact pinned versions of Node, pnpm, and libraries | 0.1 |
| Q-2 | PGlite support for `btree_gist` exclusion constraints | 3.1 |
| Q-3 | Visual design direction (style, palette, typography) | 4.1 |
| Q-4 | OpenAI model for narration | 5.3 |
| Q-5 | Full-stack hosting provider (or none) | 7.2 |
