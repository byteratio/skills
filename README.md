# Byteratio Skills

This repository is Byteratio's Claude Code **plugin marketplace**. It's where
we publish reusable [Agent Skills](https://code.claude.com/docs/en/skills) so
anyone at Byteratio can install them into their own Claude Code setup with a
couple of commands, instead of copy-pasting `SKILL.md` files between
projects.

## Adding this marketplace to your agent

In Claude Code, run:

```
/plugin marketplace add byteratio/skills
```

This registers the marketplace under the name `byteratio-skills` (the `name`
field in `.claude-plugin/marketplace.json`). List what's available and
install a plugin with:

```
/plugin list
/plugin install <plugin-name>@byteratio-skills
```

To pick up newly published skills later, refresh the marketplace's catalog:

```
/plugin marketplace update byteratio-skills
```

## Repository layout

```
skills/
├── .claude-plugin/
│   └── marketplace.json      # Catalog: lists every plugin and where to find it
├── plugins/
│   └── <plugin-name>/
│       ├── .claude-plugin/
│       │   └── plugin.json   # Plugin manifest (name, version, description, author)
│       └── skills/
│           └── <skill-name>/
│               └── SKILL.md  # The actual skill: frontmatter + instructions
└── README.md
```

Each plugin is one directory under `plugins/`. A plugin can bundle more than
one skill (add more `skills/<skill-name>/` subdirectories), but in most cases
one plugin = one skill is simplest and keeps versioning independent.

`plugins/example-skill/` is a template plugin kept in the marketplace on
purpose — copy it as a starting point rather than writing the boilerplate
from scratch.

## Adding a new skill

1. **Copy the template:**

   ```
   cp -r plugins/example-skill plugins/<your-skill-name>
   ```

2. **Rename the skill directory and edit the manifest.** In
   `plugins/<your-skill-name>/.claude-plugin/plugin.json`, set `name` to your
   plugin's kebab-case name (e.g. `deploy-helper`) and update `description`,
   `version`, and `author`.

3. **Write the skill.** Rename
   `plugins/<your-skill-name>/skills/example-skill/` to match your skill, and
   replace `SKILL.md`'s frontmatter and body:

   ```markdown
   ---
   name: your-skill-name
   description: One or two sentences stating what the skill does and,
     concretely, what the user should say or be doing for it to trigger.
   ---

   # Your Skill Name

   Instructions Claude should follow when this skill is invoked. Write this
   as directions to Claude, not as documentation about the skill.
   ```

   The `description` is what Claude matches against to decide when to
   trigger the skill — be specific about the triggering situation, not just
   what the skill does. Put any supporting scripts or reference files in the
   same directory and reference them by relative path.

4. **Register the plugin in the marketplace.** Add an entry to the `plugins`
   array in `.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "your-skill-name",
     "source": "./your-skill-name",
     "description": "What this skill does, in one line."
   }
   ```

5. **Validate before opening a PR:**

   ```
   claude plugin validate ./plugins/<your-skill-name> --strict
   claude plugin validate . --strict
   ```

6. **Open a PR.** Once merged, anyone with this marketplace added can run
   `/plugin marketplace update byteratio-skills` followed by
   `/plugin install <your-skill-name>@byteratio-skills` to pick it up.

## Conventions

- Plugin and skill names are kebab-case, no spaces.
- One plugin per directory under `plugins/`; keep the plugin name and its
  `skills/<name>/` folder name aligned so `/plugin-name:skill-name` reads
  sensibly.
- Bump a plugin's `version` in `plugin.json` when you change its behavior —
  users only get updates when the version changes.
- Don't remove or rename a published plugin without checking whether anyone
  has it installed; prefer deprecating in the description first.
