---
name: kanban-plans
description: Track multi-session plans on a locally-running kanban board (github.com/hechtch/kanban) via its agent REST API. Claim a plan, move it through todo/doing/blocked/awaiting_merge/done, leave notes, and set the git branch. Use at the start of any non-trivial multi-session plan, when resuming such a plan, or when the user asks what's on the board or where a plan stands.
---

# Plan tracking via the local kanban

The kanban exists so Claude sessions across different projects and
different days can keep plan state alive between sessions: claim
plans, move them through statuses, leave notes. Several Claude
instances may share one board — always bucket your work under a
**project slug** so plans from different repos don't collide.

When you're working on a non-trivial plan in a project on a machine
where the kanban is running, you should be talking to it as you go.

## The board

Source: **<https://github.com/hechtch/kanban>** — an Angular SPA plus
a Go API over SQLite. It runs locally; there is no hosted instance.

```sh
git clone https://github.com/hechtch/kanban.git
cd kanban
make container-build && make container-run   # CONTAINER_RUNTIME=podman if you use podman
```

`make container-run` publishes `-p 8000:8000` and mounts
`~/.kanban/data` for persistence, so a single container serves both the
UI and the API on port 8000.

| Mode | UI for the human | Agent API base |
|------|------------------|----------------|
| Container (`make container-run`) | <http://localhost:8000/board> | `http://localhost:8000/api/agent` |
| Dev (`make run`) | <http://localhost:4200/board> | `http://localhost:8000/api/agent` |

**The agent API is on port 8000 in both modes** — only the human's UI
link differs, because `make run` serves the Angular dev server
separately on 4200. Use the container's 8000 UI link unless you know
the user is running `make run`.

If the user has it deployed somewhere else (behind a reverse proxy, a
different published port), they'll tell you the base URL — take theirs
over the default and keep the `/api/agent` suffix.

### Is it up?

```sh
KB=http://localhost:8000/api/agent
curl -sS http://localhost:8000/api/health
# {"status":"ok"}
```

If that fails, the kanban isn't running. **Don't start it yourself** —
it writes to `~/.kanban/data/kanban.db` and a second process fighting
for that DB is worse than no tracking. Tell the user it's down, offer
the `make container-run` line above, and carry on with the actual work
untracked.

## Canonical spec — read this for any endpoint detail

The API ships a full **OpenAPI 3.1 spec** generated from the typed Go
handlers. It's the source of truth for paths, request/response shapes,
status codes, and validation rules — when in doubt, fetch it instead of
guessing or relying on stale docs:

- **Spec (JSON)**: `http://localhost:8000/api/openapi.json`
- **Spec (YAML)**: `http://localhost:8000/api/openapi.yaml`
- **Swagger UI** (for humans): `http://localhost:8000/api/docs`

The rest of this file gives you the *agent workflow* (slugs,
conventions, recipes) so you don't have to learn the API from scratch.
For endpoint specifics not covered here, read the spec.

Responses include a `$schema` URL field pointing at the response's JSON
schema (huma's convention). Ignore it unless you're validating
responses against the schema.

## The plan-file convention this skill assumes

The board tracks **plans**, and a plan is a markdown file in the repo:

- `plans/<topic>.md` — one file per discrete in-flight plan, describing
  the *why* and *what*. Use checkboxes (`- [ ]` / `- [x]`) for
  actionable tasks and refine them in place as work proceeds.
- `plans/done/<topic>.md` — where the file **moves** once the work
  ships or is abandoned, with a short "Delivered in \<version\> —
  \<date\>" header at the top.
- `CHANGELOG.md` — the public-facing "what changed", paired with
  `plans/done/` which carries the why.

If a repo doesn't use this layout yet, create `plans/` and seed the
file before claiming it on the board — the slug rule below depends on
the filename.

## The slug rule

Everything is keyed by **slug**, never by numeric ID. Never store a
`task.id` in your prompts or code.

- **Plan slug** = the plan filename minus `.md`, lowercased.
  `plans/photo-import.md` → `photo-import`.
- **Project slug** = the repo or initiative name in kebab-case.
  `~/projects/photo-organizer` → `photo-organizer`. (For a project
  named "Photo Organizer", the server auto-derives `photo-organizer`
  from the name; you can also pass an explicit slug.)
- Shape (enforced server-side): `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`. Bad
  slugs return `400`. Lowercase, hyphens between words, no underscores
  or capitals.

## The task body IS the plan

When you first claim a plan, **send the plan markdown as the `body`
field**. The kanban DB is canonical from that moment on — do not sync
edits back to the file mid-flight. The `plans/<slug>.md` file on disk
is the *seed* (used once); `plans/done/<slug>.md` is the *archival
snapshot* you write when the work ships.

If you re-PUT later with a new `body`, your version wins. Humans can
also fix typos in the rendered view at `/task/<task_id>` in the UI;
both edits land in the same column.

## Claiming a plan is what makes it "Claude's"

There is no assignee field to set. The board derives it: a task
carrying a `plan_slug` — which is exactly what `PUT /plans/<slug>`
creates — renders under **Claude** in the sidebar's assignee filter and
gets a Claude chip on the card. Tasks the human made by hand have no
`plan_slug` and stay under **you**.

So claiming a plan through this API is a visible act. Don't claim work
the human is doing themselves just to leave a note on it.

## Statuses

Five board columns, fixed:

```
todo  →  doing  →  blocked  →  awaiting_merge  →  done
```

Plus `backlog` (off-board, optional 6th).

- `blocked` — work can't proceed; explain why in a note.
- `awaiting_merge` — code is written and pushed, waiting for a PR to
  land or a manual merge. Different from `done` because the work hasn't
  shipped yet.
- Flip to `done` only when the branch is actually merged into the
  mainline. Don't auto-transition.

`'waiting'` is not a valid status — it was renamed to `'blocked'` in a
migration.

## Git branch

Plans (and any task) can carry a `git_branch` field — the branch where
the work is happening. Set it when you create a feature branch so the
human can see it on the card:

```sh
curl -sS -X PUT $KB/plans/photo-import \
  -H 'Content-Type: application/json' \
  -d '{"git_branch":"feat/photo-import"}'
```

Clear with `{"git_branch": null}`. The branch renders as a monospace
chip on the card.

## Suggested model & effort

Every task can carry a `model` and an `effort` — a hint for whichever
agent picks the work up next. The human sets these from the ticket (or
you can, when you know a plan's weight), and they render as a
`fable / xhigh` chip on the card. **Honour them**: when you resume a
plan whose task says `sonnet` / `high`, that's the tier the owner wants
spent on it; a `fable` / `xhigh` audit plan wants the big model at high
effort.

```sh
curl -sS -X PUT $KB/plans/security-audit \
  -H 'Content-Type: application/json' \
  -d '{"model":"fable","effort":"xhigh"}'
```

- `model` is a free-form shorthand — `fable`, `opus`, `sonnet`, `haiku`
  are the house set; the server lowercases and trims, and rejects
  anything over 40 characters.
- `effort` must be one of `low`, `medium`, `high`, `xhigh`, `max`
  (Claude Code's reasoning-effort tiers); anything else is a `422`.
- Both are optional and independent; clear with `null`. Absent means
  "no preference — use the session default".

## Projects: archiving and default tags

A project carries two fields beyond name/colour, both settable on the
same `PUT $KB/projects/<slug>` upsert:

- **`archived`** — the project is finished. Its plans drop out of the
  sidebar and off the board (folded into an "archived" section the
  human can expand), but nothing is deleted and every endpoint still
  returns them. Set it when a project's work has shipped:

  ```sh
  curl -sS -X PUT $KB/projects/photo-organizer \
    -H 'Content-Type: application/json' -d '{"archived":true}'
  ```

- **`tags`** — tags every task in the project carries. The server merges
  them into each task's `tags` on read, so a plan under a project tagged
  `["infra"]` comes back tagged `infra` without ever being tagged
  individually. Replaces the whole set, like task tags:

  ```sh
  curl -sS -X PUT $KB/projects/photo-organizer \
    -H 'Content-Type: application/json' -d '{"tags":["media","infra"]}'
  ```

  Because the merge happens on read, dropping a tag from the project
  drops it from every task at once. Don't set a project's tag on its
  tasks individually — the server strips inherited tags from a task's
  own set, so you'd just be writing a no-op.

## Workflow per session

```sh
KB=http://localhost:8000/api/agent

# 1. Make sure your project exists (idempotent — 201 first time, 200 after).
curl -sS -X PUT $KB/projects/photo-organizer \
  -H 'Content-Type: application/json' \
  -d '{"name":"Photo Organizer","color":"#d4654a"}'

# 2. Claim the plan + send its current body + put it under your project.
#    First PUT creates with status=todo; later PUTs patch.
BODY=$(jq -Rs . < plans/photo-import.md)
curl -sS -X PUT $KB/plans/photo-import \
  -H 'Content-Type: application/json' \
  -d "{\"title\":\"Photo Import\",\"project_slug\":\"photo-organizer\",\"body\":$BODY}"

# 3. Cut a feature branch and tell the kanban about it.
git checkout -b feat/photo-import
curl -sS -X PUT $KB/plans/photo-import \
  -H 'Content-Type: application/json' \
  -d '{"git_branch":"feat/photo-import"}'

# 4. Flip to doing with a note. Status PUTs are idempotent — same status
#    without a note is a no-op; with a note, writes an activity entry.
curl -sS -X PUT $KB/plans/photo-import/status \
  -H 'Content-Type: application/json' \
  -d '{"status":"doing","note":"starting Phase 2"}'

# 5. Leave notes as you go (cheaper than a status flip).
curl -sS -X POST $KB/plans/photo-import/notes \
  -H 'Content-Type: application/json' \
  -d '{"text":"phase 2 done; running tests"}'

# 6. When the PR is open, move to awaiting_merge.
curl -sS -X PUT $KB/plans/photo-import/status \
  -H 'Content-Type: application/json' \
  -d '{"status":"awaiting_merge","note":"PR #123 open"}'

# 7. Once merged, flip to done and archive the file.
curl -sS -X PUT $KB/plans/photo-import/status \
  -H 'Content-Type: application/json' \
  -d '{"status":"done","note":"merged in v0.3.0"}'
# then `git mv plans/photo-import.md plans/done/photo-import.md`
```

Mention the human's UI link when you've just created or moved a plan —
`http://localhost:8000/board` (or `:4200` in dev mode).

## Rules and gotchas

- **Status enum is fixed**: `todo`, `doing`, `blocked`,
  `awaiting_merge`, `done`, `backlog`. Don't invent new ones.
- **Unknown `project_slug` returns `422 unknown project_slug "X"`** —
  create the project before assigning. No silent auto-create.
- **`project_slug: ""`** clears the project, equivalent to
  `project_id: null`.
- **Upsert is real upsert** — `PUT /plans/:slug` with `{}` is your
  "claim if absent, patch if present" call. Re-running doesn't
  duplicate; only the first call writes a `create` activity entry.
- **Same-status status PUT is a no-op** unless you pass a `note`. Use
  this to your advantage — you can retry safely.
- **Activity log is append-only** and visible at
  `$KB/plans/<slug>/activity`. Use notes to leave breadcrumbs ("Phase 2
  done; starting Phase 3") rather than narrating in the body.
- The body is markdown rendered at `/task/<task_id>` in the UI. Use
  standard markdown: headings, lists (including `- [ ]` / `- [x]`),
  code blocks, links. Tables and blockquotes render too.

## When not to bother

- One-off bug fixes, refactors, doc tweaks. The kanban tracks **plans**
  — multi-step initiatives that span sessions or carry meaningful
  context. Single-commit work doesn't need a ticket.
- If the kanban repo is itself the project you're working on, it
  documents its own conventions in its `CLAUDE.md` — read that instead
  and ignore this file.
- If the health check fails, the board isn't running. Don't try to start
  it; just continue and tell the user.
