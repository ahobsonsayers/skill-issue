---
name: simplify-tests
description: Use when asked to simplify, clean up, or tighten tests; when tests are redundant, obvious, bloated, or brittle; or after writing new tests. Removes low-value tests and tightens survivors.
---

# Test Simplifier

## Overview

Remove redundant, obvious, low-value tests and tighten the survivors. A test earns its place only by verifying observable behavior or catching a real regression — not by restating the implementation.

Announce: "Using simplify-tests to simplify the tests."

## Scope

Default scope: **uncommitted test files only** (modified, staged, or untracked per `git status`).

- Uncommitted test files exist → work on those.
- None exist, or the user implies a broader pass → ask: all test files, or specific files/directories?
- Test files only. **Never modify source code.**

## Process

1. **Scan** — read the scoped test files (and their source when needed to judge value). Classify each test: KEEP / DELETE / TIGHTEN.
2. **Delete obvious junk immediately** (no approval gate):
   - Tautologies: mock returns X → assert X through passthrough code
   - No assertions, or only trivially-true ones
   - Exact duplicates: same code path, same branch, no new input class
   - Constants, `typeof fn === 'function'` checks, other type/existence assertions in typed languages
   - Redundant snapshots on trivial deterministic output already asserted directly
3. **Tighten survivors:**
   - Merge near-duplicates into one parametrized case
   - Delete pointless setup/teardown
   - Replace field-by-field assertion chains with one whole-object assertion
   - Remove helpers, imports, and orphaned comments left dead by deletions — verify zero remaining usages before removing any helper
4. **Verify once at end** — run the test suite. On failure, restore the last batch of changes (not the whole run) and re-check; a failing suite after cleanup is a cleanup bug, not a suite bug.

## Never delete

- The only test covering a behavior — when in doubt, KEEP
- Tests asserting external contracts (payment, API, filesystem) or critical side effects (rollback, cleanup)
- Tests that document distinct behavior even if covered elsewhere

## Red flags

- Deleting something because it's hard to test, not because it's low-value
- Touching production code to make cleanup easier
- Batching deletions so large a failure can't be attributed to a batch

Report briefly: what was deleted, what was tightened, test results.