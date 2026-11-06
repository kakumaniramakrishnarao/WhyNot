---
name: whynot-review
description: Use to independently review a finished WhyNot session (a PR or branch diff) against its session card and AGENTS.md. Run in a fresh task. Produces findings; does not modify code unless explicitly asked.
---

# Independent session review

Inputs: the session ID and the PR/branch. Read `AGENTS.md`, the session card, and the docs it lists. Then review the diff (`git diff main...HEAD`).

## Checklist — report every failed item as a finding

**Scope**
- [ ] Every "Build" item is implemented; nothing from "Out of scope" or later sessions was added.
- [ ] Only expected files changed (card's file list + tests + PROGRESS/QUESTIONS).

**Correctness**
- [ ] Each "Done when" item is actually satisfied — find the test or evidence for each.
- [ ] Logic matches the spec docs exactly (rule semantics, buffer definition, cost model, projection rules, verifier checks, API status codes).
- [ ] Boundaries handled: half-open intervals, DST, exact-equality edges, empty inputs.

**Invariants (AGENTS.md)**
- [ ] Engine purity (no forbidden imports/globals), determinism (canonical ordering), epoch-minute time.
- [ ] Contracts defined only in `packages/contracts`.
- [ ] Patient surfaces only receive projected data; no secrets client-side.
- [ ] Dependency direction respected; no lint rules disabled.

**Tests**
- [ ] New behaviour has tests; tests assert meaningful outcomes (not just "no throw").
- [ ] No `.skip`/`.only`/`todo`, no lowered run counts, no weakened assertions, no golden expectation edits without a decision-log entry.
- [ ] `pnpm verify` passes (run it).

**Quality**
- [ ] Names, file layout and conventions match AGENTS.md; no dead code, no stray debug output, no `any`.
- [ ] UI sessions: loading/empty/error states, accessibility rules, design tokens used, screenshots attached.

## Output

A list of findings, most severe first: `[blocker|major|minor] file:line — problem — why it matters — suggested fix`. End with a verdict: **merge**, **merge after minor fixes**, or **changes required**. If asked to post on the PR, use one review with inline comments.
