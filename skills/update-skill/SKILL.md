---
name: update-skill
description: >
  Use at the end of any conversation where a meaningful insight was gained about improving
  a skill — record durable, generalizable lessons (content improvements, structural changes,
  effectiveness tweaks) in the skill's SKILL.md. Do NOT use to record task-specific notes
  unless they reflect a broader improvement to the skill itself. If nothing durable was
  learned, make no changes. Even if the user just says "fix this skill" or "improve this
  skill."
keywords: [skill improvement, update skill, skill.md, skill authoring, skill maintenance]
---

# Update Skills

Review the conversation for any durable, generalizable lesson about improving the last invoked skill.

## Rules

- **Only update if** a durable lesson was learned: a principle, process, debugging heuristic, or pattern that would make the skill more effective.
- **Skip if** the lesson is specific to a single task and doesn't generalize to the skill's purpose.
- **One concise addition** under the appropriate section of the skill's SKILL.md. Add a new section only if necessary.
- **If nothing durable was learned, make no changes.**

## Process

1. Determine which skill to update:
   - If a skill was explicitly mentioned in the request, update that skill.
   - Otherwise, update the last skill invoked in this conversation.
   - If unclear which skill is meant, ask the user.
2. Review the conversation for any generalizable insight about improving that skill.
3. If found, read the skill's SKILL.md to understand its current structure.
4. Apply the improvement — one concise addition under the appropriate section.
5. **Validate:** re-read the skill's SKILL.md to confirm the edit was applied correctly. If the edit is missing or malformed, fix and re-validate.
6. **REQUIRED:** Use the [writing-skills](../writing-skills/SKILL.md) skill for the full TDD-based methodology on how to structure, test, and bulletproof skill content. This skill handles the *what* (which skill to update, when); writing-skills handles the *how* (format, testing, loophole-closing).
