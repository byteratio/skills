---
description: Analyze all changes and commit in logical groups.
model: sonnet
---

# Commit

## Step 1: Survey changes

Then run the commit-survey to get to get the file lists and classification. It lives beside this file, not in the repo you are mapping, so resolve its path first:

> **`SKILL_DIR`** — the absolute path of the directory containing _this
> SKILL.md_, which your harness reported when it loaded this file. It differs
> per tool (`~/.claude/skills/commit`,
> `~/.codex/skills/commit`, `~/.agents/skills/commit`, a
> plugin cache, or a project-local `.claude/skills/…`). Substitute the literal
> path; do not rely on an environment variable.

Run `$SKILL_DIR/bin/commit-survey` to get the file lists and classification.

Read diffs of key files if you need more context on the changes.

## Step 2: Group the changes

Use the `--- classified ---` output as a starting point, then refine into logical commits.

Grouping strategy:

- By domain/feature: e.g., all auth changes together
- By layer: e.g., model tests, controller tests
- By type: e.g., all config changes, all dependency updates

Files in `[skip]` should NOT be committed. If unsure about a file, skip it.

## Step 3: Commit each group

For each group, combine stage + commit in one call:

git add <specific files> && git commit -m "<message>"

Order commits from most independent to most dependent:

- Config/tooling changes first
- Then source code changes
- Then test changes
- Generated files last

Attribute each commit to the author of the changes, DO NOT not claim authorship yourself. Use `--author` if needed.
