# 07 — Roadmap: phases, sessions & gates

A **session** = one self-contained unit of work that ends with a working, tested, pushed increment (one feature, or a clean half of a big one). Time doesn't matter; the *done-when* checklist does. Each session has a card in [`docs/sessions/`](sessions/) with exact scope, files, tests, and a starter prompt for Codex.

Phases are ordered so the **riskiest thing (the engine's correctness and speed) is proven first**, before any UI exists. A phase's exit gate must pass before the next phase starts.

## Phase overview

| Phase | Goal | Sessions | Exit gate |
| --- | --- | --- | --- |
| **0 — Foundation** | Monorepo, tooling, CI, guard rails | 0.1, 0.2 | `pnpm verify` green locally and in CI; import-boundary lint works; skills installed |
| **1 — Engine core (proof)** | Correct, fast feasibility + minimal explanations | 1.1 – 1.6 | P1–P5, P8, P9, P12 pass at 5,000 runs; H1–H8 explanation sets pass; S-dataset bench within targets |
| **2 — Alternatives, admin fix, wording** | Complete engine output + template wording | 2.1 – 2.4 | P6, P7, P10, P11 pass; H1–H8 fully pass incl. alternatives and patient text; every rule has templates |
| **3 — API & data** | Persistence, seed clinic, endpoints, safe booking | 3.1 – 3.4 | Integration suite green on PGlite; concurrency test passes; seed reproduces hero scenarios through the API |
| **4 — Patient experience** | The demo: booking page + "Why not?" panel + static mode | 4.1 – 4.5 | E2E for H1, H2, H3, H7 pass in full and static mode; axe clean; static build deploys |
| **5 — AI narration** | Verified OpenAI wording | 5.1 – 5.3 | Verifier adversarial suite passes; narration eval ≥ 90% first-pass, 0 privacy failures; model choice recorded |
| **6 — Staff tools** | Staff view, insights, simulator, inspector | 6.1 – 6.4 | Dashboard populated from seed; simulator estimates match a hand-checked example; inspector shows H1 matrix |
| **7 — Ship** | Hardening, deploy, portfolio package | 7.1 – 7.3 | Release checklist (below) 100% ticked |

**Total: 31 sessions.**

## Session index

| ID | Session | Depends on |
| --- | --- | --- |
| 0.1 | Monorepo scaffold, tooling, `pnpm verify`, CI | — |
| 0.2 | Guard rails: import boundaries, engine purity lint, PR template, skills check, ADR template | 0.1 |
| 1.1 | Contracts: zod schemas for snapshot, request, fact, evaluation | 0.2 |
| 1.2 | Time module: wall-clock ↔ instant, intervals, DST tests | 1.1 |
| 1.3 | Rule catalog + global/patient/timing rules | 1.2 |
| 1.4 | Practitioner, room, equipment rules (incl. buffers/turnover) | 1.3 |
| 1.5 | Candidates, violation matrix, feasibility, reference impl, differential tests | 1.4 |
| 1.6 | Explanation (min-weight hitting set) + properties + Riverside fixtures + H1–H8 explanation goldens + bench → **Phase 1 gate** | 1.5 |
| 2.1 | Alternatives search + cost model + diversity | 1.6 |
| 2.2 | Admin fix + presentation groups + `evaluate()` final shape | 2.1 |
| 2.3 | Audience projection (privacy) + P10 | 2.2 |
| 2.4 | Template narrator (all rules, both audiences) + edge goldens → **Phase 2 gate** | 2.3 |
| 3.1 | DB schema, migrations, PGlite test harness, Docker Compose | 2.4 |
| 3.2 | Seed script: Riverside clinic + demo clock + historical evaluations | 3.1 |
| 3.3 | API skeleton + snapshot loader + `/clinic`, `/availability`, `/explain` | 3.2 |
| 3.4 | Bookings (serializable + exclusion constraints), cancel, policies, demo reset → **Phase 3 gate** | 3.3 |
| 4.1 | Design system with UI UX Pro Max + web scaffold + API client + role switcher | 3.4 |
| 4.2 | Booking page: service/practitioner/date pickers + availability grid | 4.1 |
| 4.3 | "Why not?" panel + alternatives + one-click book + conflict handling | 4.2 |
| 4.4 | Static demo mode (engine in browser) + GitHub Pages workflow | 4.3 |
| 4.5 | E2E + accessibility pass + responsive polish → **Phase 4 gate** | 4.4 |
| 5.1 | Narrator interface + OpenAI narrator + cache + budget | 4.5 |
| 5.2 | Verifier + adversarial tests + fallback wiring in API/UI | 5.1 |
| 5.3 | Narration eval runner + model choice + static narrations → **Phase 5 gate** | 5.2 |
| 6.1 | Staff view of explanations + admin fix display + policy editor | 5.3 |
| 6.2 | Insights dashboard | 6.1 |
| 6.3 | Policy simulator | 6.2 |
| 6.4 | Engine inspector (matrix visualisation) → **Phase 6 gate** | 6.3 |
| 7.1 | Hardening: security review, rate limits, error states, perf recheck | 6.4 |
| 7.2 | Deploy + environments + runbook | 7.1 |
| 7.3 | Portfolio package: README, GIF, diagram, video script, write-up → **Release** | 7.2 |

Phases 1 and 2 are split finely on purpose: the engine is where mistakes are most expensive. Small sessions (e.g. 1.2) may be done back-to-back in one sitting, but each is still committed, verified and pushed separately.

## Release checklist (v1.0)

- [ ] All phase gates passed and recorded in `docs/PROGRESS.md`
- [ ] `pnpm verify` + CI e2e green on `main`
- [ ] H1–H8 + ≥ 16 edge goldens pass; helpfulness review scores ≥ 4
- [ ] Narration eval report committed; 0 privacy failures
- [ ] Bench within targets; baseline committed
- [ ] Security review (skill `security-best-practices`) findings resolved or documented
- [ ] Static demo live; full mode runs with one command (`docker compose up` + `pnpm dev`)
- [ ] README with GIF, architecture diagram, limitations; write-up published in `docs/writeup.md`
- [ ] LICENSE present; no secrets in history; `.env.example` complete

## Change control

- Scope changes → add an entry to [08 — Decision log](08-decisions.md) first, then edit the affected docs and session cards, then code.
- If a session reveals the spec is wrong or ambiguous: **stop**, write the question in `docs/QUESTIONS.md`, and finish only the unambiguous part. Never guess silently.
