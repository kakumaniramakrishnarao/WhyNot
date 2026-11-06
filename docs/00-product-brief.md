# 00 — Product brief

## One sentence

WhyNot is an explainable booking engine that, when a booking request fails, returns the minimal set of rules that block it, the nearest alternatives that would work, and — for staff — which policies are costing the business bookings.

## The problem

Booking systems answer a failed request with "unavailable". That hides three different situations:

1. **One obvious reason** (clinic closed on Sunday).
2. **Several rules that only block in combination** (the room is free, but the only two qualified practitioners are, respectively, inside a cleaning buffer and on leave).
3. **A policy, not a physical limit** (a 24-hour notice rule blocks a slot that is otherwise completely free).

Patients guess, give up, or contact support. Staff dig through settings to reconstruct why. Nobody learns which rules quietly cost bookings every week.

## Where WhyNot lands (and where it doesn't)

Most booking apps simply **hide** unavailable times. When a patient has plenty of open slots that suit them, that works fine — they pick one, and **WhyNot adds little there. We don't claim otherwise.**

Hiding fails in three moments, and those are WhyNot's three surfaces:

| Moment | What goes wrong with hidden slots | What WhyNot does |
| --- | --- | --- |
| **The patient is stuck** — a whole day, a whole week, or their preferred practitioner shows *nothing*, or the time they need is missing | They can't tell "fully booked" from "on leave", "needs a referral", "returning patients only" or "2 hours' notice". Many causes are *rules*, which a hidden slot can never communicate. They give up or phone the clinic | **"Why is this empty?"** for a day or practitioner (the most common real moment), plus "why not this time?" for a specific slot, and always "here's what you *can* book" |
| **The front desk gets the call** | Staff reconstruct the cause from calendars and settings | One click: the full staff-level reason and the minimal policy fix |
| **The manager never finds out** | A hidden slot leaves no trace; the clinic never learns which rules cost it bookings | Every "no" is recorded with its reason → insights ("the 15-min buffer hid 41 requested times") + a what-if simulator |

In one line: **normal apps hide the "no"; WhyNot explains it when the patient is stuck, answers it for staff, and counts it for the business.** The third surface is the strongest value; the first is the most visible in the demo.

## Users

| User | What they need | WhyNot surface |
| --- | --- | --- |
| **Patient** (booking online) | "Why can't I book this, and what can I book instead?" in plain, polite language that never exposes other people's information | Booking page + "Why not?" panel |
| **Front-desk / clinic admin** | The full reason (including staff-only facts like leave), plus what policy change would have allowed it | Staff view of the same explanation |
| **Clinic manager** | Which rules block the most requests; what happens if a policy changes | Insights dashboard + policy simulator |
| **Developer / reviewer** (portfolio audience) | Proof the engine is correct and the AI cannot invent reasons | Engine inspector, test suite, write-up |

## What already exists (checked November 2026)

| Existing thing | What it does | Gap WhyNot targets |
| --- | --- | --- |
| Cal.com "Why unavailable?" troubleshooter | Host-side; shows the *main* reason a slot is blocked and a suggested setting fix | One reason, host-only, no multi-resource interaction, no ranked alternatives, no aggregate insights |
| CozyCal, NeetoCal, Meetergo, etc. | Help-centre articles / troubleshooters listing common causes | Static guidance, not computed explanations |
| WhyUnsat, clingo-explaid, TRACE-CS (research) | Minimal explanations of unsatisfiable constraint problems | Research artifacts; no product UX, not booking-specific |
| AI support agents (Pluno, Pylon, Intercom Fin) | LLM investigates tickets using logs/DBs | The LLM decides; explanations are not provably correct |

**Positioning:** WhyNot combines rigorous constraint explanation (minimal blocking sets + counterfactual alternatives) with a polished, privacy-aware product experience, and uses AI only for wording that is verified against the engine's output. The unique value is the **combination** — no single feature is unprecedented.

## What makes it good (the bar we hold ourselves to)

1. **Correct by construction.** Every "why" is checkable: the stated reasons are sufficient to block the request, and each is necessary. Every suggested alternative is re-evaluated and bookable. These are tested properties, not hopes.
2. **Helpful, not just correct.** The explanation is the *simplest* sufficient one (weighted by how meaningful each reason is to the reader), and alternatives are ranked by how little they change the patient's request.
3. **Honest AI.** The model never decides. It rewords structured facts; a verifier rejects uncited or invented content and falls back to templates.
4. **Private.** Patients never see other patients, leave reasons, or internal notes. Staff see more. Visibility is enforced in the explanation data, not in the UI.
5. **Runs free.** Template mode and a static in-browser demo mean zero hosting or model cost for reviewers.

## Scope

### In scope (v1.0)

- Single-appointment requests at one clinic (one timezone).
- Resources: practitioners (skills, hours, breaks, leave), rooms (capabilities, maintenance), equipment (types, maintenance).
- Rule families: availability, occupancy, buffers/turnover, capability, timing policy (notice, advance window), daily caps, patient eligibility, patient's stated preferences.
- Outputs: bookable assignment *or* minimal explanation + up to 3 diverse alternatives + staff-only "minimal policy fix".
- **Range explanations:** "Why is this day / this practitioner's week empty?" — a headline common cause (if one exists), time segments with their reasons, and the next available slot.
- Demo booking flow (books into the demo database), staff insights, policy simulator, engine inspector.
- Template wording + optional OpenAI wording with verification.
- Static in-browser demo (GitHub Pages) + full-stack local mode (Docker Compose).

### Out of scope (deliberately)

- Real calendar sync (Google/Outlook), payments, reminders, notifications.
- Multi-clinic / multi-tenant SaaS, real authentication, real patient data.
- Recurring/series bookings and group bookings (the algorithm design notes how they would extend).
- Integrating with third-party schedulers (Cal.com etc.) — a possible later adapter, not part of v1.0.
- Letting the AI make or change bookings.

## Success criteria

| Area | Target |
| --- | --- |
| Correctness | All engine properties (doc 06) pass on ≥ 5,000 generated cases per CI run and all golden scenarios |
| Helpfulness | All 8 hero scenarios produce the explanation and top alternative a human reviewer expects |
| AI honesty | 100% of displayed AI wording passes the verifier (by construction); 0 privacy leaks across the narration eval |
| Speed (demo dataset) | Explain p95 ≤ 150 ms, alternatives p95 ≤ 400 ms in Node on a laptop |
| Demo | A first-time visitor understands the value in < 60 s, with no signup or key |
| Portfolio | README with GIF, architecture diagram, 2-minute video, technical write-up, honest limitations |

## Main risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Looks like Cal.com's troubleshooter | Hero scenarios are multi-rule interactions; build those first (Phase 1 gate) |
| Correct but unhelpful explanations | Weighted minimum explanation + reviewer rubric on hero scenarios |
| "Nearest alternative" feels arbitrary | Explicit, configurable cost model; diversity across alternative types |
| Performance | Benchmark in Phase 1 before any UI; algorithm designed around a small candidate matrix |
| Scope creep | Fixed out-of-scope list; every new idea goes to the decision log first |
| LLM cost / misbehaviour | Template default, cache, verifier, budget limits, static demo uses pre-verified narrations |

## Honest limitations (to publish)

- The engine explains the rules it knows. If a real clinic has unwritten rules, explanations are correct about the model, not about reality.
- "Nearest alternative" reflects a stated preference model; a different patient may prefer differently.
- Policy simulation is an estimate: it replays past requests independently and ignores knock-on effects (an extra accepted booking would have occupied time others later wanted).
