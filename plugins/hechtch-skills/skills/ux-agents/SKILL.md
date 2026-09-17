---
name: ux-agents
description: Scaffold persona-based UX testing agents for a web app — project-scoped .claude/agents/<persona>-ux.md files that drive a real browser via Playwright, walk user journeys as a specific persona, and report tagged friction findings. Use when setting up UX review automation, when the user asks what a specific user segment would think of the app, or after the user accepts an offer to add persona testing.
---

# UX testing agents — persona-based browser-driven review

A pattern for catching UX issues that escape unit tests and human
self-review: define **persona agents** that drive a real browser via
Playwright, walk through journeys from a specific user's perspective,
and report friction. Real-world result that proved this useful: an
agent flagged a workspace-dropdown desync bug in the same session it
first ran, before anyone had filed it.

## When to set this up

- Project has a web frontend with **multiple distinct user types** —
  newgrad vs. mid-career vs. retiree, free-tier vs. enterprise admin,
  patient vs. provider, etc. The personas need to differ enough that
  defaults / vocabulary / available features actually feel different
  to each one.
- The app has enough surface area that walking it manually for every
  persona on every change isn't realistic.
- Dev server can be run locally without auth gymnastics (or auth has a
  scriptable bypass for testing).

Skip this for: APIs without a UI, CLIs, single-persona internal tools.

## Where the agents live

Project-scoped at `.claude/agents/`, **not** in the user's global
config. Personas embed project-specific knowledge (numeric ranges,
feature names, journeys) that wouldn't generalize. Each persona is its
own `.md` file with frontmatter:

```markdown
---
name: <persona-slug>-ux
description: UX-tests <app> from the perspective of a <one-line persona>. Drives a real browser via Playwright, screenshots each tab, and reports friction points. Use proactively after UI changes affecting <area>.
tools: Bash, Read, Write, Edit
---

You are a UX testing agent role-playing a <persona>.

## Your persona
<5–10 lines: age, role, knowledge level, goal in this session, what
they would NOT know>

## Workflow
1. Create a fresh workspace via API (don't pollute real data).
2. Seed inputs via API to match the persona.
3. Drive the browser in /tmp/ux-<slug>.mjs with Playwright.
4. Screenshot the tabs relevant to this persona.
5. Read each screenshot and evaluate.
6. Write findings to plans/ux-reports/<slug>-<date>.md.
7. Clean up the test workspace.
```

Also drop a `_shared-ux-protocol.md` in the same directory (the leading
underscore keeps it from being mistaken for an agent). That file holds
environment expectations, output format, the "what to evaluate" rubric,
etc. — the per-persona files quote it where useful.

## Required scaffolding

- **`plans/ux-reports/`** exists (create it during setup so reports have
  somewhere to land). This assumes the convention that `plans/` in the
  repo holds working documents about in-flight work, committed to git so
  the reasoning outlives the chat session; UX reports are one such
  document. If a project has no `plans/` directory, create it — or agree
  another location with the user and write it into every persona file,
  since the agents hardcode the path.
- Playwright works via `npx playwright` (agents may need
  `npx playwright install chromium` once on a fresh machine — bake that
  into the agent's instructions).
- An API for seeding data must exist, OR a deterministic UI flow must be
  documented. Without one of these, the agent ends up too brittle.

## Persona design rules

- **Knowledge gap is the value** — the newgrad agent finds jargon the
  experienced one wouldn't blink at; the retiree finds the
  oversimplifications. If two personas would give identical reports,
  collapse them.
- **State a concrete goal per session**, not a vague "explore the app".
  "Can I retire someday on this salary?" is testable. "Use the app" is
  not.
- **Specify what they DON'T know** — "doesn't know what SWR / Coast FI /
  72(t) mean" forces the agent to flag those terms rather than gloss
  past them.
- **Numeric ranges matter** — if the persona has $5K saved, defaults
  designed for $500K savers will surface bugs in chart axis scaling,
  zero-handling, etc. that median personas would miss.

## Output discipline (encode in every agent)

- Tag every finding `[blocker]` / `[friction]` / `[polish]`.
- One-line suggested fix per issue.
- End with a "Top 3 fixes to ship next" section so the user gets a
  prioritized action list.
- Report only what was actually observed in screenshots / API responses
  — no speculative complaints.

## Running an agent

After dropping the `.md` files, **restart Claude Code** — agent
definitions are scanned at session start. From the new session: `use the
<persona>-ux agent to review the calculator after my latest changes`.
Each run takes minutes (real browser), so don't blast them on every
change — run after meaningful UI work.

## What to track for the user

When setting this pattern up, note in the project's `CHANGELOG.md` or a
plan doc under `plans/`:

- That the agents exist and where the reports go.
- The dev-server prerequisite (so a new contributor doesn't try to run an
  agent against a cold project).
- Any project-specific data-seeding quirks the personas rely on.

## Common pitfalls

- **Don't have agents start the dev server.** They tend to leave zombie
  processes. Have them check `curl -fsS http://localhost:<port>` and
  refuse to proceed if it's down — the user starts it.
- **Don't reuse a real workspace / dataset.** Always create-and-delete a
  `UX: <persona>` workspace. Findings polluted by leftover state waste
  the user's review time.
- **Don't make the persona a generic "casual user".** That's where the
  noise comes from. The more specific the persona, the more useful the
  report.
- **Don't run too many in parallel** — Playwright + dev server +
  Chromium per agent will OOM a laptop fast. Serial runs are fine.
