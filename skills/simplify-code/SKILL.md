---
name: simplify-code
description: Use when asked to simplify, clean up, refactor, or tighten code, or to strip unneeded comments; or after writing or changing code that may be more complex than needed. Preserves behavior. Defaults to uncommitted changes, falls back to the whole branch vs main when nothing is uncommitted, and can be told to do the whole branch.
---

# Simplify Code

Refine code for clarity and maintainability without changing behavior. Readable and explicit beats short and clever. Follow the project's own conventions (`AGENTS.md`, linter config) and the [code-style](../code-style/SKILL.md) skill.

## Scope

Pick the scope in this order:

1. **Override:** the user asks for the whole branch → all changes on the branch vs `main`, plus uncommitted changes. A scope the user names (files, directories) wins over everything.
2. **Default:** uncommitted changes in the working tree (modified, staged, untracked per `git status`).
3. **No uncommitted changes, on a branch other than `main`:** the whole branch vs `main`. List files with `git diff --name-only $(git merge-base HEAD main)`. Use `master` or `origin/HEAD` if `main` does not exist.
4. **No uncommitted changes, on `main`, or no git:** ask the user what to simplify.

Do not mix refactoring with feature or bug-fix work.

Skip when:
- The code is already clean. Do not simplify for its own sake.
- You do not yet understand what it does.
- The simpler version would be measurably slower in performance-critical code.

## Process

1. Find the code in scope.
2. Before changing or removing anything, find out why it exists: what calls it, what edge cases it handles, whether an old constraint still applies (Chesterton's Fence). If unclear, read more context first.
3. Make one simplification at a time. Run tests after each. If they fail, revert that change.
4. Confirm the result is simpler, not just different.

## What to simplify

- Remove needless nesting, redundant code, and abstractions that do nothing.
- Use clear names. Consolidate related logic.
- No nested ternaries. Use `if`/`else` or `switch`.
- Keep helpful abstractions and separation of concerns. Do not merge unrelated concerns to save lines.

## Comments

Go through EVERY file in scope, including Dockerfiles and config files, and delete comments that narrate the code: what it does, step labels, package justifications, history. Keep a comment only if the code would invite a wrong change without it (non-obvious constraint, counter-intuitive workaround, trap). When unsure, delete. Git log owns history.

State the present constraint, never the rejected alternative. Write "must accept ECDSA keys", not "previously X" or "we chose A over B". Rejected alternatives belong in decision docs (`AGENTS.md`), not code.

## Red flags — stop and reconsider

- A simplification needs test changes to pass. You likely changed behavior.
- The "simplified" code is longer or harder to follow.
- You rename to match your taste, not project convention.
- You remove error handling to make code look cleaner.
- You batch many simplifications into one hard-to-review change.
- You edit code outside the task scope.
