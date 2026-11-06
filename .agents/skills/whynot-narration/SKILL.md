---
name: whynot-narration
description: Use for WhyNot wording work — per-rule templates (patient and staff), the Narrator interface, the OpenAI narrator (Responses API, structured output, cache, budget), the verifier, and the narration eval runner. Use with the openai-docs skill for current API syntax.
---

# Wording and AI narration in WhyNot

Spec: `docs/04-explanation-and-ai.md` (authoritative), `docs/06-testing-and-accuracy.md` (narration eval).

## Templates

- One function per rule ID per audience in `packages/narrator/src/templates/`; plus group templates.
- Templates only use `fact.params`, subject `displayName`, and formatting helpers (clinic timezone, locale). No engine calls, no data lookups.
- Patient tone: polite, specific, never blaming, never revealing staff-only info. Staff tone: precise, includes rule names and settings.
- The catalog-coverage test enforces both templates for every rule.

## OpenAI narrator

- Check current API syntax with `openai-docs` before coding (Responses API + strict JSON schema output). Do not rely on memory for parameter names.
- Model from `OPENAI_MODEL`; key from `OPENAI_API_KEY`; both read only in `apps/api` config and passed in. The narrator package never reads `process.env` itself.
- Input = projected facts + template sentences + alternatives. Never pass unprojected evaluations for patient audience.
- Timeouts, one transport retry, no retry on verification failure, cache by content hash, daily cap.
- Bump `PROMPT_VERSION` whenever the system prompt or few-shots change.

## Verifier

- Deterministic, no model calls. Checks: schema, citations exist, coverage, token grounding, privacy, no promises, length (doc 04 table).
- Any failure ⇒ return template narration with `source: 'template'` and the report.
- Every check has adversarial unit tests (fake model outputs that should fail): invented time, invented name, missing citation, staff-only word, promise phrase, other patient's name, extra sentence count.
- Prefer false rejections over false acceptances. Don't loosen a check to raise pass rates — improve the prompt instead.
