---
name: whynot-ui
description: Use for any work in apps/web of WhyNot — pages, components, design system, availability grid, Why-not panel, staff views, insights charts, static demo mode, accessibility and responsive polish. Use together with the ui-ux-pro-max skill for visual design decisions.
---

# Working on the WhyNot web app

Spec: `docs/01-architecture.md` (web structure), `docs/04-explanation-and-ai.md` (what to display), `docs/design/` (design system from Session 4.1), session card.

## Design

- Visual decisions (palette, typography, spacing, component styles, chart styling) come from the design system created in Session 4.1 with `ui-ux-pro-max` and saved in `docs/design/`. Use its tokens (Tailwind theme) — no ad-hoc colours or font sizes.
- Before inventing a new pattern, consult `ui-ux-pro-max` (UX guidelines + pre-delivery checklist) and reuse existing components in `src/components/`.
- Tone: calm, trustworthy healthcare. Explanations must be scannable: reasons as a short list with icons + text, alternatives as large tappable buttons.

## Rules

- The UI **displays** engine output; it never computes, filters, or reorders reasons or alternatives (formatting only).
- Data fetching via TanStack Query hooks in `src/api/` generated around `@whynot/contracts` schemas; parse responses with zod.
- Every async view has loading, empty, error and success states.
- Show wording source: "Plain wording" vs "AI wording — verified". Show "Demo — fictional clinic" and the demo clock in the header.
- Patient view must never display staff-only information; it only receives projected data — don't add workarounds.
- Static mode (`VITE_MODE=static`): same components, data from `src/static/` adapter running the engine in-browser. Never branch business logic on mode inside components — only the data adapter differs.

## Accessibility (WCAG 2.2 AA target)

- All interactive elements keyboard reachable with visible focus; logical tab order.
- Availability grid: buttons with accessible names ("Tuesday 14:30, unavailable — see why").
- The Why-not panel is a dialog/drawer with focus management, `aria-live="polite"` for results.
- Status never by colour alone (icon + text). Contrast ≥ 4.5:1 text.
- axe check in e2e: no serious/critical issues.

## Before finishing a UI session

Run the app, check desktop (1280px) and mobile (390px) widths, take screenshots with the `screenshot`/`playwright` skill, and attach them to the PR description.
