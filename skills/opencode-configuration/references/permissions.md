# Permissions Reference

Add or update a shell permission rule in the top-level `permissions` array of `~/.config/opencode/opencode.jsonc`. OpenCode V2 replaced the V1 `permission.bash` glob-keyed object with an **ordered array** of `{action, resource, effect}` rules; the last matching rule wins. Shell commands use `action: "shell"` (V1 called this `"bash"`). Resources use the same glob syntax as before: a trailing ` *` matches any arguments, a bare resource matches the no-arg form. See the [permissions docs](https://opencode.ai/v2/docs/permissions).

<Callout>
`permissions` also holds every other action (`read`, `edit`, `skill`, `subagent`, MCP tool actions, etc.), not just shell commands. This reference is scoped to the shell-command section of that array — the same section V1 called "bash permission rules."
</Callout>

## Multiple Config Files — Check for Shadowing First

If the user has more than one config file at the same tier (e.g. `~/.config/opencode/opencode.jsonc` plus a second file for machine-local overrides), **do not assume their `permissions` arrays merge.** Confirmed empirically: when two documents each define a top-level `permissions` key, one array **replaces** the other wholesale in the effective policy — it does not concatenate the way `mcp.servers`, `agents.<id>`, and `plugins` do across the same files.

Symptom: the user reports being asked for permission on commands that are clearly `allow`ed in their main config. Before touching any rules, check:

1. `find ~/.config/opencode -maxdepth 1 -name "opencode*.json*"` — is there more than one config file?
2. If so, run `opencode debug config` **from inside a real project directory** (not the config directory itself, and not a scratch/temp dir) and inspect each returned document's own `permissions` array — don't assume the main file's array is the one actually in effect.
3. Cross-check `~/.local/share/opencode/opencode.db`'s `permission` table for saved "Allow always" approvals: `sqlite3 -header -column ~/.local/share/opencode/opencode.db "SELECT action, resource, datetime(time_created/1000,'unixepoch') FROM permission ORDER BY time_created DESC LIMIT 40;"`. Repeated saved approvals for commands that are already allowed in the main config, especially across several projects, are a strong signal that a second file's `permissions` array is shadowing it.

**Fix:** keep exactly one file as the sole source of the `permissions` array. Move any rules from the other files into it, and delete the `permissions` key from every other config file — even a short array there will still shadow the main one.

## Shell Rule Structure

The shell-command rules follow a broad default for the `shell` action, then are grouped under category headers (`// CategoryName`). Within each category, entries are **alphabetical by command name** (the `resource` value up to its first space).

Reuse the categories already in the user's file. Do not reflow entries between categories; only sort within a category.

Example shape:

```jsonc
"permissions": [
  { "action": "shell", "resource": "*", "effect": "ask" },

  // Core utilities
  { "action": "shell", "resource": "cat *", "effect": "allow" },
  { "action": "shell", "resource": "echo *", "effect": "allow" },
  { "action": "shell", "resource": "sort *", "effect": "allow" },

  // Git
  { "action": "shell", "resource": "git diff *", "effect": "allow" },
  { "action": "shell", "resource": "git push *", "effect": "ask" },
],
```

## Required Steps

1. **Identify the category.** Pick the existing category the command naturally belongs to. If it could fit two, pick the more specific one (e.g. `kubectl get` → Kubernetes, not Core utilities). If it fits none, **ask the user** which category to use or whether to add a new one — do not silently drop it into a catch-all.
2. **Pick the rule's `effect`.** `allow` / `ask` / `deny`. For a read-only command, default to `allow`. For a mutating or destructive command, default to `ask`. If unsure, `ask`.
3. **Insert alphabetically within the category** by command name (`cat` < `echo` < `sort` < `test` < `tr`). Preserve the trailing ` *` and match the file's existing indentation.
4. **Do not reorder existing entries** unless the user explicitly asks to reorganize. Only insert the new entry at the correct alphabetical position.
5. **Insert after the broad `shell` default, not before it.** Because `permissions` is an ordered array where the last match wins, a new specific rule only takes effect if it appears after the broad `{ "action": "shell", "resource": "*", "effect": "ask" }` default. Keep new entries inside the category sections, never above the default rule.
6. **If the user's shell wraps commands** (an alias or hook that rewrites `cat` to something else), check what the command actually runs as and add a rule for that literal string too. Test with the wrapper's own dry-run or rewrite command; do not guess from its help text.

## Common Mistakes

| Mistake | Fix |
| ------- | --- |
| Writing `"bash": {"cmd *": "allow"}` | V2 uses `{ "action": "shell", "resource": "cmd *", "effect": "allow" }` inside the `permissions` array. |
| Adding a permission as an object key instead of an array entry | `permissions` is an ordered array of `{action, resource, effect}` objects, not a glob-keyed object. |
| Inserting a new rule before the broad `shell: * → ask` default | Last match wins — specific rules must come after the broad default to override it. |
| Permission rule in the wrong category | Place under the matching category header; pick the more specific one if two fit. |
| Breaking alphabetical order within a category | Sort by command name within the category (the `resource` value up to its first space). |
| Silently dumping an uncategorized command into a catch-all category | Ask the user which category to use, or whether to add a new one. |
| Reordering existing entries while inserting a new one | Only insert the new entry; do not reflow existing categories. |
| Assuming `permissions` merges across multiple config files | It replaces wholesale, not merges. Keep `permissions` in exactly one file (see "Multiple Config Files" above). |

## Red Flags — STOP and Fix

- "I'll add a `\"cmd *\": \"allow\"` key like the old docs said." → V2 uses `{ "action": "shell", "resource": "cmd *", "effect": "allow" }` array entries, not object keys.
- "It fits no existing category, so I'll put it in a catch-all." → Ask the user which category to use first.
- "I'll insert my new rule at the very top of the array for safety." → It must come after the broad `shell: * → ask` default or the last-match-wins ordering will never apply it.
- "It's just a permission line, no need to validate." → Run prettier + schema validation on every change (see SKILL.md Finalizing).
- "The user has a second config file too, but `permissions` will just merge across both." → It won't. Check with `opencode debug config` from a real project dir first (see "Multiple Config Files" above).
