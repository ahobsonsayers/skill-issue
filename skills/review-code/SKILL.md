---
name: review-code
description: Use when the user asks for a code review of uncommitted changes, a commit, a branch or a pull request, such as "review my changes", "review this", "code review", "review PR 42" or "is this ready to merge", or types /review-code.
---

# Review Code

You orchestrate. A fresh reviewer subagent reviews, a fresh fixer subagent fixes, and the user picks each next step. Nothing loops on its own.

Never review or fix the code yourself, even when the diff is small or you wrote it. A fresh reviewer is not anchored on the reasoning that produced the change, and your context stays free to run the rounds. Do not read the diff either: `git status` and `git diff --stat` are enough to pick the scope.

If you cannot dispatch subagents, say so and ask the user whether to run the review in this session instead. If they agree, follow the reviewer and fixer prompts yourself.

The reviewer uses [simplify-code](../simplify-code/SKILL.md), [simplify-tests](../simplify-tests/SKILL.md) and [code-style](../code-style/SKILL.md) as checklists. Pull requests need the GitHub CLI (`gh`).

## 1. Pick the scope

First match wins. Do not ask when one matches.

1. The user names files, commits, a branch or a PR: review that. Diff: see "Named scopes" below.
2. Uncommitted changes exist (`git status --porcelain`, untracked files included): review only those, even on a feature branch. Diff: `git diff HEAD`.
3. On a branch other than the default branch: review the whole branch. Diff: `git diff $(git merge-base HEAD <default>)`.
4. On the default branch with nothing uncommitted: ask the user for a branch or PR, then stop.

The default branch is `origin/HEAD` (for example `origin/main`). If it is not set, use `main` or `master`. If neither exists, ask.

### Named scopes

- Files: `git diff $(git merge-base HEAD <default>) -- <files>`. If that is empty, put "none: review these files whole: <files>" in the Diff slot, and keep it for every round.
- Commits: `git diff <oldest commit>^`, for example `git diff HEAD~3` for the last 3 commits. If later commits or uncommitted changes exist, this diff includes them: ask the user whether to include them. If not, use `git diff <oldest commit>^ <newest commit>`.
- A branch or PR: the fixer edits the working tree, so the branch must be checked out. If it is not, ask before you run `git switch <branch>` or `gh pr checkout <pr>`. With uncommitted changes, do not check out: ask the user to commit or stash them, then stop. Diff: rule 3. The base is the default branch, or `origin/<baseRefName>` for a PR.

For a PR, first run `gh pr view <pr> --json number,title,body,headRefName,baseRefName` and `git fetch origin <baseRefName>`.

## 2. Write the change summary

Write 2 to 5 lines on what the change should do: its goal and requirements. Leave out how it was built and why you think it works, because that anchors the reviewer. Sources, best first:

1. This session, if the changes were made here.
2. The PR title and body. `gh pr view --json title,body` with no number finds the PR for the current branch.
3. The messages of the commits in scope: `git log --format='%s%n%b' <base>..HEAD`. Uncommitted changes have none.

If none exist, write "Unknown. Infer it from the diff."

## 3. Dispatch the reviewer

Dispatch one fresh subagent with this prompt. Fill in the `<slots>` and copy the rest word for word.

~~~text
Review a code change. Report only: do not edit files, commit, stash or switch branches.

Repo: <absolute path>
Scope: <what is reviewed, for example "uncommitted changes" or "branch feat/x vs origin/main">
Diff: run `<diff command>`, and read the untracked files from `git ls-files --others --exclude-standard`
What the change should do: <change summary>
Earlier findings: <"none", or one line each: <title> (<path>): fixed | not chosen | skipped, <reason>>

1. Read the diff, then the code around it: the callers of each changed function, what it calls, its tests and the README. When a signature or behavior changes, find every caller.
2. Read the project rules: AGENTS.md and CLAUDE.md in the repo root and in the folders of the changed files.
3. Load all 3 of these skills before step 4, and use them as checklists only. Skip every step in them that edits files. If one cannot be loaded, say so under "Not checked".
   - code-style
   - simplify-code
   - simplify-tests
4. Look for:
   - Bugs: wrong results, crashes, edge cases (empty, zero, last item, rounding, time zones), and callers the change breaks.
   - Risk: injection, leaked secrets, unchecked input, swallowed errors, and writes that can lose data.
   - Load: races, unbounded growth, a query per item, and slow code on big inputs. Judge against the expected load in the README or config. Name the load you assumed in each load finding.
   - Tests: risky new logic with no test that fails when it breaks.
   - Rules: project rules the change breaks. Quote the rule.
   - Simplicity: what the 3 skills flag, dead code, and code that an existing helper or the standard library already covers.
5. Check each finding before you report it. Re-read the lines and name a concrete case: this input or situation gives this wrong result. No case, no finding. Drop:
   - problems the change did not add
   - what a linter or type checker catches
   - taste, and anything you would phrase as "consider"
   - earlier findings marked "not chosen"
   Check each earlier finding marked "fixed". Report a broken fix as a new finding.

Severity:
- Must fix: a bug, a security hole, data loss, or a break at the expected load.
- Should fix: risky logic with no test, real slowness, a broken project rule, flaky tests, dead code the change added, or logic an existing helper or library already does (name it).
- Nits: findings that only change how the code reads, such as naming, formatting, comments and needless nesting. Never higher.

Reply with only this report. Use plain English, and explain a term the first time you use it.

Verdict: <Ship. | Fix <numbers> first.>
Scope: <what you reviewed, in one line>
What this change does: <1-2 sentences>

**Must fix**
1. **<title>** (`<path>:<lines>`)
   Problem: <the concrete case>
   Fix: <the smallest fix; prefer deleting code>

**Should fix**
2. ...

**Nits**
3. ...

Not checked: <one line, only if something that matters could not be checked>

Number the findings across all groups. Leave out empty groups. With no findings, write "Verdict: Ship.", the scope, and one line on what you checked.
~~~

## 4. Show the report and ask what to fix

Show the report as the reviewer wrote it. If it has no findings, stop.

Ask the user which findings to fix, with these options:

- Must fix and should fix (recommended; if there are only nits, recommend Done)
- All, including nits
- Done

The user can also type numbers, for example "1 and 3". Fix nothing before the user answers.

## 5. Dispatch the fixer

Dispatch one fresh subagent with this prompt. Fill in the `<slots>` and copy the rest word for word.

~~~text
Fix these review findings in <absolute repo path>.

Diff: run `<diff command>`
What the change should do: <change summary>
Findings to fix, copied from the review:
<the chosen findings, word for word>

- Fix only these findings, with the smallest change. Touch nothing else, and do not offer other changes.
- For a nit from code-style, simplify-code or simplify-tests, load that skill and follow its rules.
- For a bug, add a test that fails without the fix.
- If a finding is wrong or unsafe to fix, skip it and say why.
- Do not commit, push, stash or switch branches.
- At the end, run the project's tests, and its linter if it has one.

Reply with one line per finding: "<number>: fixed (<files>)" or "<number>: skipped, <reason>". Then give the test command and its result.
~~~

## 6. Show the fixes and ask to review again

Show the fixer's reply. Ask the user: review again (recommended), or done.

To review again, go back to step 3 with the same scope. The diff command must include the uncommitted fixes: if it has a file filter, an end commit or no diff, also give `git diff HEAD`. Fill in "Earlier findings" with every finding so far and its status. Repeat until the user says done or a review finds nothing.

## Common mistakes

- Reviewing inline because the diff is small, you already read it, or other instructions prefer fewer subagents. Dispatch the reviewer anyway.
- Reviewing the whole branch when there are uncommitted changes.
- Ranking a finding that only changes how the code reads above Nits.
- Fixing before the user picks, or starting another review without asking.
- Committing the fixes.
