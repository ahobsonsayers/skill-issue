# Permissions Reference

Add or update a shell permission rule in the top-level `permissions` array of `~/.config/opencode/opencode.jsonc`. OpenCode V2 replaced the V1 `permission.bash` glob-keyed object with an **ordered array** of `{action, resource, effect}` rules; the last matching rule wins. Shell commands use `action: "shell"` (V1 called this `"bash"`). Resources use the same glob syntax as before: a trailing ` *` matches any arguments, a bare resource matches the no-arg form. See the [permissions docs](https://opencode.ai/v2/docs/permissions).

<Callout>
`permissions` also holds every other action (`read`, `edit`, `skill`, `subagent`, MCP tool actions, etc.), not just shell commands. This reference is scoped to the categorized shell-command section of that array — the same section V1 called "bash permission rules."
</Callout>

## Multiple Config Files — Check for Shadowing First

If the user has more than one config file at the same tier (e.g. `~/.config/opencode/opencode.jsonc` plus a companion `~/.config/opencode/opencode.local.jsonc` for personal/machine-local overrides), **do not assume their `permissions` arrays merge.** Confirmed empirically: when two documents each define a top-level `permissions` key, one array **replaces** the other wholesale in the effective policy — it does not concatenate the way `mcp.servers`, `agents.<id>`, and `plugins` do across the same files.

Symptom: the user reports being asked for permission on commands/actions that are clearly `allow`ed in their main config. Before touching any rules, check:

1. `find ~/.config/opencode -maxdepth 1 -name "opencode*.json*"` — is there more than one config file?
2. If so, run `opencode debug config` **from inside a real project directory** (not the config directory itself, and not a scratch/temp dir) and inspect each returned document's own `permissions` array — don't assume the main file's array is the one actually in effect.
3. Cross-check `~/.local/share/opencode/opencode.db`'s `permission` table for saved "Allow always" approvals: `sqlite3 -header -column ~/.local/share/opencode/opencode.db "SELECT action, resource, datetime(time_created/1000,'unixepoch') FROM permission ORDER BY time_created DESC LIMIT 40;"`. Repeated saved approvals for commands that are already in the main config's allowlist — especially across several different projects — is a strong signal that a second file's `permissions` array is shadowing it.

**Fix:** keep exactly one file as the sole source of the `permissions` array. Move any "local-only" rules into it, and delete the `permissions` key entirely from every other config file — even a short array there will still shadow the main one.

## Shell Rule Structure

The shell-command portion of the `permissions` array is organized in **two halves**, each with category headers (`// CategoryName`), following the broad default rule for the `shell` action:

1. **Native block** — native binaries, grouped by category. Categories appear in this fixed order: Core utilities, Text processing, Files/directories, Archives, Checksums, Network, Miscellaneous tools, Git, AWS, Docker, Github, Kubernetes, Helm, Go, …
2. **RTK block** — preceded by a `// RTK equivalents to above commands` divider, mirroring the same category order as the native block.

Within each category, entries are **alphabetical by command name** (the `resource` value up to its first space). Do not reflow entries between categories; only sort within a category.

Example shape:

```jsonc
"permissions": [
  { "action": "shell", "resource": "*", "effect": "ask" },

  // Core utilities
  { "action": "shell", "resource": "cat *", "effect": "allow" },
  { "action": "shell", "resource": "echo *", "effect": "allow" },
  { "action": "shell", "resource": "sort *", "effect": "allow" },
  { "action": "shell", "resource": "test *", "effect": "allow" },

  // ...
  // -------------------------------------------
  // ---- RTK equivalents to above commands ----
  // -------------------------------------------

  // Core utilities
  { "action": "shell", "resource": "rtk diff *", "effect": "allow" },
  { "action": "shell", "resource": "rtk grep *", "effect": "allow" },
  { "action": "shell", "resource": "rtk read *", "effect": "allow" },
  // ...
],
```

## Required Steps

1. **Identify the category.** Pick the category the command naturally belongs to (see the category list above). If a command could fit two categories, pick the more specific one (e.g. `kubectl get` → Kubernetes, not Core utilities). If it fits no existing category, **ask the user** which category to use or whether to add a new one — do not silently drop it into "Miscellaneous tools".
2. **Pick the rule's `effect`.** `allow` / `ask` / `deny`. For a read-only command, default to `allow`. For a mutating or destructive command, default to `ask`. If unsure, `ask`.
3. **Insert alphabetically within the category** by command name. `cat` < `echo` < `sort` < `test` < `tr`. Preserve the trailing ` *` and 4-space indentation (one level inside the top-level `permissions` array — not the deeper 6-space nesting V1 used for its object form).
4. **Add the rtk equivalent** when one exists. `rtk --help`'s subcommand list is NOT the source of truth — some commands not listed there (e.g. `task`, `make`, `df`, `du`, `stat`, `jq`) still get silently rewritten by rtk's hook layer, while most unlisted commands (e.g. `echo`, `mkdir`, `atlas`, `sort`, `which`) do not. **Always check with `rtk rewrite "<cmd> <example-args>"`:**
   - Non-empty output (e.g. `rtk rewrite "task lint"` → `rtk task lint`, `rtk rewrite "cat foo.txt"` → `rtk read foo.txt`) means that literal rewritten string is what actually executes — add a rule matching it exactly in the **RTK block**, same category, alphabetical: `{ "action": "shell", "resource": "rtk <cmd> *", "effect": "..." }`.
   - Empty output means the command runs unchanged (plain, no `rtk` prefix) — skip the RTK block entirely, do not add a speculative `rtk foo` or `rtk proxy foo` entry.
   - Never use the `rtk proxy <cmd>` form for mirrors — that's a distinct explicit rtk subcommand, not what the rewrite hook actually produces. If `rtk rewrite` shows a rewrite, it's always a bare `rtk <cmd> ...`, never `rtk proxy <cmd> ...`.
5. **Do not reorder existing entries** unless the user explicitly asks to reorganize. Only insert the new entry (and its rtk mirror) at the correct alphabetical position.
6. **Insert after the broad `shell` default, not before it.** Because `permissions` is an ordered array where the last match wins, a new specific rule only takes effect if it appears after the broad `{ "action": "shell", "resource": "*", "effect": "ask" }` default. The category sections already live after that default — keep new entries inside them, never above the default rule.

## Common Mistakes

| Mistake                                                                                            | Fix                                                                        |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Writing `"bash": {"cmd *": "allow"}`                                                                | V2 uses `{ "action": "shell", "resource": "cmd *", "effect": "allow" }` inside the `permissions` array. |
| Adding a permission as an object key instead of an array entry                                     | `permissions` is an ordered array of `{action, resource, effect}` objects, not a glob-keyed object. |
| Inserting a new rule before the broad `shell: * → ask` default                                      | Last match wins — specific rules must come after the broad default to override it. |
| Permission rule in wrong category                                                                  | Place under the matching category header; pick the more specific one if two fit. |
| Permission rule breaking alphabetical order within its category                                    | Sort by command name within the category (the `resource` value up to its first space). |
| Adding `rtk foo` speculatively based on `rtk --help`'s subcommand list                             | Check `rtk rewrite "<cmd> args"` instead — it's the real source of truth, and catches unlisted rewrites (`task`, `make`, `df`, `du`, `stat`, `jq`, …) that `--help` misses. |
| Mirroring with `rtk proxy <cmd>` form                                                              | Rewrites always produce bare `rtk <cmd> ...`, never `rtk proxy <cmd> ...`. Only use `rtk proxy` if the user explicitly wants that literal invocation. |
| Adding the rtk mirror to the native block instead of the RTK block                                 | rtk rules live in the RTK block under the same category, alphabetical.    |
| Silently dumping an uncategorized command into "Miscellaneous tools"                              | Ask the user which category to use, or whether to add a new one.           |
| Reordering existing entries while inserting a new one                                              | Only insert the new entry; do not reflow existing categories.             |
| Assuming `permissions` merges across multiple config files (e.g. main + `.local.jsonc`)            | It replaces wholesale, not merges. Keep `permissions` in exactly one file (see "Multiple Config Files" above). |

## Red Flags — STOP and Fix

- "I'll add a `\"cmd *\": \"allow\"` key like the old docs said." → V2 uses `{ "action": "shell", "resource": "cmd *", "effect": "allow" }` array entries, not object keys.
- "It fits no existing category, so I'll put it in Miscellaneous tools." → Ask the user which category to use first.
- "I'll add an `rtk <cmd>` rule to be safe." → Only if `rtk rewrite "<cmd> args"` actually produces a rewritten string; `--help`'s subcommand list is not reliable enough on its own.
- "`rtk --help` doesn't list it, so no rtk mirror needed." → Wrong test. Verify with `rtk rewrite` — several commands (`task`, `make`, `df`, `du`, `stat`, `jq`) rewrite silently despite not appearing in `--help`.
- "I'll insert my new rule at the very top of the array for safety." → It must come after the broad `shell: * → ask` default or the last-match-wins ordering will never apply it.
- "It's just a permission line, no need to validate." → Run prettier + schema validation on every change (see SKILL.md Finalizing).
- "The user has a `.local.jsonc` too, but `permissions` will just merge across both." → It won't. Check with `opencode debug config` from a real project dir first (see "Multiple Config Files" above).
