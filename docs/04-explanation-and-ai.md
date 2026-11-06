# 04 — Explanations, privacy & AI narration

## The Explanation contract (v1)

Defined in `packages/contracts/src/evaluation.ts` (zod). Abbreviated:

```ts
Fact = {
  id: string                 // stable, e.g. "practitioner-buffer:prac=dr-rao:booking=bk-0142"
  ruleId: RuleId             // catalog id, doc 02
  ruleVersion: number
  kind: 'availability' | 'occupancy' | 'capability' | 'timing' | 'patient' | 'preference' | 'policy'
  subject: { type: 'clinic' | 'patient' | 'practitioner' | 'room' | 'equipment' | 'request', id: string, displayName: string }
  params: Record<string, string | number>   // e.g. { bufferMin: 15, adjacentStart: "14:00", adjacentEnd: "14:30" }
  window?: { from: Instant, to: Instant }   // the time span that conflicts, if any
  visibility: 'public' | 'staff'
  relaxableBy: ('patient' | 'admin')[]
}

Explanation = {
  primary: FactId[]          // minimum-weight hitting set (doc 03), ordered for presentation
  groups: FactGroup[]        // presentation grouping, see below
  perCandidateSummary: { practitionerId: string, factIds: FactId[] }[]
  optimal: boolean           // false if B&B budget hit and greedy fallback used
  facts: Record<FactId, Fact>
}

Alternative = { request: BookingRequest, assignment: Assignment, category: 'same-day' | 'other-practitioner' | 'other-day',
                cost: number, changes: { field: 'start' | 'practitioner' | 'date', from: string, to: string }[] }

AdminFix = { factIds: FactId[], changes: { setting: string, from: string | number, to: string | number }[], viaCandidate: Assignment }
```

The contract is versioned (`EXPLANATION_SCHEMA_VERSION = 1`). Any breaking change bumps it and gets a decision-log entry.

### Presentation grouping

The engine groups primary facts for readability (deterministic rules, in `engine/src/explain.ts`):

| Group | Contains | Example rendering |
| --- | --- | --- |
| `request` | timing, clinic, patient facts | "Appointments need at least 2 hours' notice." |
| `per-practitioner` | one group per qualified practitioner, their blocking facts | "Dr. Rao — has a 15-minute buffer after a 14:00–14:30 appointment." |
| `capability-context` | all `practitioner-skill` facts collapsed | "Only Dr. Rao and Dr. Mehta offer minor procedures." |
| `resources` | room/equipment facts | "The ultrasound unit is under maintenance until 12:00." |

## Audience projection (privacy)

`projectForAudience(evaluation, 'patient' | 'staff')` is the **only** way an evaluation reaches a patient-facing surface. It is pure, in the engine, and heavily tested.

Rules for `patient`:

1. Facts with `visibility: 'staff'` are **replaced** by a generic public fact of the same subject: `practitioner-time-off` → "Dr. Mehta is unavailable at that time." The `reason` param is dropped.
2. Occupancy facts never include other patients' IDs, names, or services — only the busy window ("booked 14:00–14:30") or for room/equipment just "in use".
3. `room-capability` and `equipment-type` facts are dropped unless *no* room/unit qualifies, in which case one public fact "This service isn't available at this clinic site" remains.
4. `adminFix` and `perCandidateSummary` are removed; `trace` is removed.
5. Fact IDs in patient output are **opaque** (hashed per evaluation), so internal booking IDs never leak.

A property test asserts: for any evaluation, the serialized patient projection contains no booking IDs, no patient IDs other than the requester's, no `reason` values, no staff-only rule IDs.

## Template wording (the default)

`packages/narrator/src/templates/` holds one template per rule ID per audience, plus group templates. Templates are plain TypeScript functions `(fact, ctx) => string` using the clinic's locale and timezone formatting.

- Every rule in the catalog **must** have both templates; a test iterates the catalog to enforce it.
- Template output is the **ground truth wording**: it is what the UI shows without AI, and what the verifier compares against.
- Tone guide (patients): polite, specific, never blaming ("That time is within…" not "You chose…"), always followed by alternatives.

## AI narration (optional polish, OpenAI)

### Purpose

Turn the grouped template sentences into one short, natural paragraph for patients, and a crisp summary for staff. **The model adds no information.** If it can't be verified, we show templates.

### Interface

```ts
interface Narrator {
  narrate(input: NarrationInput): Promise<Narration>
}
NarrationInput = { audience: 'patient' | 'staff', facts: Fact[] /* already projected */, groups: FactGroup[],
                   alternatives: Alternative[], templateSentences: { factIds: FactId[], text: string }[], locale: string }
Narration = { sentences: { text: string, factIds: FactId[] }[], source: 'ai' | 'template',
              model?: string, promptVersion: string, verification: VerificationReport }
```

Implementations: `TemplateNarrator` (always available) and `OpenAINarrator` (server only).

### OpenAI usage

- Responses API with a **strict JSON schema** output: `{ sentences: [{ text, factIds[] }] }`.
- Model from env `OPENAI_MODEL` (choose the cheapest current small model that passes the narration eval in Session 5.3; record the choice in the decision log). Key from env `OPENAI_API_KEY`, server-side only.
- System prompt (versioned `PROMPT_VERSION`): role, tone guide per audience, the hard rules (only use given facts, cite factIds on every sentence, never promise, never mention other people, keep times exactly as given), and 3 few-shot examples.
- Input is the **projected** facts + template sentences — the model never sees staff facts when narrating for patients.
- Limits: `max_output_tokens` ~300, timeout 8 s, 1 retry on transport errors only, no retry on verification failure (fall back instead).
- Cache: key = SHA-256 of (projected facts, alternatives, audience, locale, model, PROMPT_VERSION). Stored in `narrations` table. Identical explanations never cost twice.
- Budget: per-IP rate limit and a daily global call cap (env `NARRATION_DAILY_CAP`); when exceeded → templates.

### Verifier (`packages/narrator/src/verify.ts`)

Runs on every AI narration. Any failure ⇒ `source: 'template'` and the report is logged.

| Check | Rule |
| --- | --- |
| Schema | Output parses against the JSON schema |
| Citations exist | Every `factIds` entry exists in the input; every sentence cites ≥ 1 fact |
| Coverage | Every primary fact (or its group) is cited by at least one sentence; alternatives mentioned only if provided |
| Token grounding | Every time (`14:30`, `2:30 pm`), date, weekday, duration/number, and proper name found in a sentence must appear in the rendered params/templates of the facts **that sentence cites**, or in a provided alternative. Extracted with deterministic regexes + the known-names list from the snapshot |
| Privacy | For patient audience: no staff-only vocabulary list ("leave", "sick", "holiday", "maintenance contractor"…), no names other than the requester, practitioners, clinic and rooms/equipment in cited facts |
| No promises / no invented actions | Deny-list: "guarantee", "we will", "I've booked", "refund", "exception", "waitlist" (unless a future feature provides them) |
| Length | ≤ 4 sentences, ≤ 90 words (patient); ≤ 6 sentences (staff) |

The verifier is deliberately conservative: false rejections only cost a little fluency; false acceptances would break trust.

### Static demo

The static (GitHub Pages) build ships **pre-generated, verified narrations** for the hero scenarios (produced by the eval runner), so visitors see AI wording without any key. Other requests in static mode use templates. The UI labels the wording source ("Plain wording" / "AI wording — verified").
