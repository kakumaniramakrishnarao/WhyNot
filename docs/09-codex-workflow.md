# 09 — Working with Codex

This project is built session by session with OpenAI Codex. The plan assumes **Codex is a capable but fallible implementer**: every session is tightly specified, every result is verified by tests and CI, and an independent review pass checks the work before it merges.

## How Codex reads this repo

| File / folder | Loaded | Purpose |
| --- | --- | --- |
| `AGENTS.md` (root) | Automatically, every run (32 KiB limit) | Standing rules: architecture invariants, commands, conventions, what never to do |
| `apps/web/AGENTS.md`, `packages/engine/AGENTS.md` | Automatically when working in those folders | Area-specific rules (added in Sessions 0.2 and 4.1) |
| `.agents/skills/<name>/SKILL.md` | When the task matches the skill description, or explicitly via `$name` | Task procedures (session protocol, adding a rule, UI work, review…) |
| `docs/sessions/phase-*.md` | When the prompt points to them | The exact scope of a session |

Restart Codex after changing `AGENTS.md` or skills (it builds the instruction chain once per run).

## One-time setup

1. **Toolchain** (local machine): Node 22 LTS, `corepack enable` (provides pnpm), Git, Docker (only for local Postgres demo; not needed for tests).
2. **Codex CLI or IDE extension** signed in. For cloud tasks, create a Codex environment for this repo with this setup script:
   ```bash
   corepack enable
   pnpm install --frozen-lockfile
   pnpm exec playwright install --with-deps chromium
   ```
   (Before Session 0.1 there is no lockfile; the first session runs `pnpm install` without `--frozen-lockfile`.)
3. **Install curated OpenAI skills** (once, user scope). In Codex, run each:
   ```
   $skill-installer playwright
   $skill-installer playwright-interactive
   $skill-installer screenshot
   $skill-installer security-best-practices
   $skill-installer security-threat-model
   $skill-installer openai-docs
   $skill-installer gh-fix-ci
   $skill-installer gh-address-comments
   $skill-installer yeet
   ```
   Then restart Codex. (Names from `github.com/openai/skills` → `skills/.curated`, checked Nov 2026. If one has been renamed, run `$skill-installer` with no argument to list them.)
4. **Install UI UX Pro Max** (design intelligence skill, MIT, `github.com/nextlevelbuilder/ui-ux-pro-max-skill`) **inside the repo** during Session 4.1:
   ```bash
   npm install -g ui-ux-pro-max-cli
   uipro init --ai codex
   ```
   It needs Python 3 for its search scripts. Commit the generated skill folder so cloud tasks have it. Check where it was installed (expected under `.agents/skills/`), and if it landed elsewhere, record the path in `AGENTS.md`.
5. **GitHub:** protect `main` (require PR + CI green). Codex works on branches and opens PRs.

## Skills map: which skill for which work

Project skills (in this repo, written for WhyNot):

| Skill | Use it for |
| --- | --- |
| `$whynot-session` | **Every session.** Start-of-session reading, plan, implement, verify, hand-off |
| `$whynot-engine` | Any change in `packages/engine` or `packages/contracts` (rules, matrix, explain, alternatives) |
| `$whynot-testing` | Writing property, differential, golden, integration and e2e tests |
| `$whynot-api` | Routes, DB schema, migrations, transactions in `apps/api` |
| `$whynot-ui` | Anything in `apps/web` (works together with `ui-ux-pro-max`) |
| `$whynot-narration` | Templates, OpenAI narrator, verifier, narration eval |
| `$whynot-review` | Independent review of a finished session's diff (run in a **fresh** Codex task) |

External skills:

| Skill | Sessions |
| --- | --- |
| `ui-ux-pro-max` | 4.1 (design system), 4.2–4.5, 6.1–6.4, 7.3 (screenshots/GIF polish) |
| `playwright` / `playwright-interactive` | 4.3, 4.5, 6.x e2e; manual exploration of the running app |
| `screenshot` | 4.5, 7.3 (README images) |
| `security-threat-model` | 3.3 (draft), 7.1 (final) |
| `security-best-practices` | 3.4, 5.1, 7.1 |
| `openai-docs` | 5.1–5.3 (current Responses API + structured output syntax, model list) |
| `gh-fix-ci` | Whenever CI is red on a PR |
| `gh-address-comments` | After the review pass leaves PR comments |
| `yeet` | End of each session: stage, commit, push, open PR |

## The session loop

```
1. Pick the next unchecked session in docs/PROGRESS.md.
2. Implementation task (Codex, reasoning effort: high for Phases 1–2, medium elsewhere):
      paste the session's Starter prompt.
3. Codex works on branch  session/<id>-<slug>, runs `pnpm verify`, updates PROGRESS + handoff, opens a PR ($yeet).
4. Review task (NEW Codex task, fresh context):
      "$whynot-review Review PR for session <id> against its card in docs/sessions/<file>.md.
       Report findings; do not modify code."
5. If findings: back in the implementation task → "$gh-address-comments" / fix → re-verify.
6. You check the "Human check" list below, then squash-merge.
```

### Human check before merging (2 minutes)

- [ ] CI green (verify + e2e when applicable).
- [ ] Diff only touches files the session card lists (plus tests/docs). Unexpected files → ask why.
- [ ] No tests deleted, skipped (`.skip`, `.only`, `todo`) or weakened; no golden expectation changed without a decision-log entry.
- [ ] `docs/PROGRESS.md` ticked with a short hand-off note.
- [ ] For UI sessions: open the preview and look at it. For engine sessions: read the golden test output.

### When Codex goes wrong

| Symptom | Do this |
| --- | --- |
| Changes things outside scope | Reject; re-run with "Only touch files listed in the session card." |
| Tests "fixed" by editing expectations | Reject the PR. Re-prompt citing the AGENTS.md golden rule |
| Stuck after 2 attempts at the same failure | Stop. Ask it to write the failing case + hypothesis in `docs/QUESTIONS.md`, then resolve it in a planning task (below) |
| Spec seems wrong | It must stop and write in `docs/QUESTIONS.md` — never improvise architecture |
| CI red only in GitHub | `$gh-fix-ci` |

## Starter prompt template

Every session card has a prompt in this shape. Copy it as-is:

```
$whynot-session
Session <ID> — <title>.
Read AGENTS.md, docs/PROGRESS.md and docs/sessions/<file>.md (section "Session <ID>") fully before writing code.
Also read: <docs listed in the card>.
Use skills: <skills listed in the card>.
Work on branch session/<ID>-<slug>. Implement exactly the "Build" list; nothing from "Out of scope".
Finish only when every "Done when" item is true and `pnpm verify` is green.
If anything in the spec is ambiguous or contradictory, stop and write it in docs/QUESTIONS.md instead of guessing.
End by updating docs/PROGRESS.md and opening a PR with $yeet.
```

## Planning tasks (when a question or spec change comes up)

There is no separate planning tool after the initial plan. When `docs/QUESTIONS.md` has an open item, or you want to change scope:

```
Planning task — no code changes.
Read AGENTS.md, docs/00–09 and docs/QUESTIONS.md item <#>.
Give me 2–3 options with trade-offs and a recommendation, consistent with the architecture invariants.
After I choose, update docs/08-decisions.md, the affected docs and session cards, and docs/QUESTIONS.md (mark resolved).
Open a PR titled "plan: <short description>".
```

Merge that PR before continuing the build. Code always follows docs.
