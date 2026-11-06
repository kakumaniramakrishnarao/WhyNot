# WhyNot

**An explainable booking engine: when a booking can't be made, WhyNot tells you exactly why — and the smallest change that would make it work.**

> Status: **planning complete, implementation not started.** Everything in `docs/` is the build plan. No code exists yet.

Every booking system says "slot unavailable" and stops there. WhyNot answers the real questions:

- **Why not?** The *minimal set of rules* that together block the request — including rules that only block in combination ("Dr. Rao has a 15-minute buffer after her 2:30 procedure, and Dr. Mehta — the only other practitioner qualified for this — is on leave").
- **What would work?** The nearest valid alternatives, computed and re-verified by the engine ("2:45 with Dr. Rao", "3:00 with Dr. Mehta tomorrow").
- **Why is this whole day empty?** Most apps just hide unavailable times — fine until a patient finds *nothing* for the day, the week, or their doctor. WhyNot explains the empty screen ("Ultrasound scans need a referral", "Dr. Okafor doesn't work Fridays — next: Monday 10:00").
- **What is this costing us?** For clinic staff: which rules block the most requests, and what would happen if a policy changed.

The decision is always made by a **deterministic rules engine**. An AI model (OpenAI) only rewords the engine's structured result into friendly language — and a verifier rejects any sentence that doesn't cite a real reason. With no API key, WhyNot uses built-in templates, so the demo runs at zero cost.

The demo business is **Riverside Family Clinic** — fictional practitioners, rooms, equipment, patients and policies.

## Documentation map

| Doc | What it answers |
| --- | --- |
| [00 — Product brief](docs/00-product-brief.md) | What we build, for whom, what exists already, what makes it unique, what's out of scope |
| [01 — Architecture](docs/01-architecture.md) | Components, packages, data flow, runtime modes, tech stack |
| [02 — Domain & rule catalog](docs/02-domain-and-rules.md) | The clinic model and every rule the engine knows, with IDs |
| [03 — Engine algorithms](docs/03-engine-algorithms.md) | Feasibility, minimal explanations, alternatives, admin fixes, determinism, performance |
| [04 — Explanations & AI narration](docs/04-explanation-and-ai.md) | Explanation contract, privacy, templates, OpenAI narrator, verifier |
| [05 — API & data](docs/05-api-and-data.md) | REST endpoints, database schema, demo clock and reset |
| [06 — Testing & accuracy](docs/06-testing-and-accuracy.md) | How correctness is *proven*, how helpfulness is judged, benchmarks |
| [07 — Roadmap](docs/07-roadmap.md) | Phases, exit gates, what "done" means for the whole project |
| [08 — Decision log](docs/08-decisions.md) | Every decision, why, and what was rejected |
| [Sessions](docs/sessions/) | Session-by-session build cards with starter prompts |
| [PROGRESS](docs/PROGRESS.md) | Live checklist of sessions |
| [09 — Working with Codex](docs/09-codex-workflow.md) | Skills to install, session loop, review, what to do when the agent goes wrong |
| [AGENTS.md](AGENTS.md) | Rules every coding session follows (read automatically by Codex) |

## Planned stack

TypeScript everywhere · pnpm monorepo · pure-TS engine (runs in browser and Node) · Fastify API · PostgreSQL + Drizzle · React + Vite · OpenAI for optional wording · Vitest, fast-check, Playwright.
