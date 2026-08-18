---
name: example-skill
description: Template skill for this marketplace. Use as a starting point when adding a new skill — copy this plugin directory, rename it, and replace this description with the specific trigger conditions for your skill (what the user says or does that should invoke it).
---

# Example Skill

This is a placeholder skill used only to demonstrate the expected layout for
plugins in this marketplace. Replace this file's contents with the actual
instructions for your skill.

## Guidance for real skills

- Keep the frontmatter `description` specific: state what the skill does and,
  concretely, what user request or situation should trigger it. Claude
  matches skills against this description, so vague text causes missed or
  spurious triggers.
- Write the body as instructions to follow when the skill is invoked, not as
  documentation about the skill.
- Put any supporting scripts, templates, or reference files alongside
  `SKILL.md` in this same directory and reference them by relative path.
