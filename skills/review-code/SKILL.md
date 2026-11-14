---
name: review-code
description: Use when the user asks for a code review of uncommitted changes, a commit, a branch or a pull request, such as "review my changes", "review this", "code review", "review PR 42" or "is this ready to merge", or types /review-code.
---

# Review Code

You orchestrate, and you apply the fixes. A fresh reviewer subagent reviews; this session holds the context, decides what gets fixed, applies the fixes and runs the rounds. Nothing loops on its own.

The owner sees one message from you before the first review. It states the budget, the scope and the change summary, and ends with the counters line:

```
up to 2 fix rounds (max 3 reviewer passes); nits are listed at the end, never fixed in bulk
Scope: <the paths in scope>  (diff: <the diff command>)
Change summary: <2 to 5 lines on what the change should do>
review budget: reviews 0/3, fix rounds 0/2 | next: review 1
```

Every later message that leads to a step ends with the same counters line, with that step already counted and the next step named. See "Loop budget".

Never review the change yourself, even when the diff is small or you wrote it. A fresh reviewer is not anchored on the reasoning that produced the change.

You do apply the fixes, so you read the files you edit, and only those, and only to apply the chosen findings. Do not read the whole diff, do not read the files you are not editing, and do not probe the behaviour of the change to form your own opinion of it: a second opinion from you is how a review starts fixing things nobody asked for.

If you think the reviewer missed something, do not fix it. Name it in the final message as an observation for the owner, not as a finding, and do not spend a round on it.

If you cannot dispatch subagents, say so and ask the user whether to run the review in this session instead. If they agree, follow the reviewer prompt yourself.

The reviewer works from the diff, the code around it and the repo's own rules. It loads no other skill. Pull requests need the GitHub CLI (`gh`).

## Pipeline order

1. The simplify pass ([simplify-code](../simplify-code/SKILL.md) and [simplify-tests](../simplify-tests/SKILL.md)) belongs to the owner, not to the review. Say in your first message whether the change has been through it. If it has not, recommend it in one line and carry on. Do not run it yourself, and do not load those skills in the reviewer.
2. Review → fix → scoped re-review → fix → scoped re-review. Two fix rounds, no more.
3. Nits are listed once in the final message. Commit only when the owner asks.
4. The handoff goes to the plan ledger, or to a scratch file outside the repo when there is no ledger. See step 7.

Shape findings from the reviewer are nits during a review: shape is the simplify pass's job. The reviewer finds nits on its own, without a checklist skill.

## Loop budget

```
fix rounds       : max 2
reviewer passes  : max 3   (the first review, plus one after each fix round)
findings that may trigger a round : must-fix and should-fix only (nits never do)
terminal states  : Ship | Accepted trade-offs | Owner decision required
```

The dispatch order is fixed. There are six steps and no others:

```
1. review 1
2. fix round 1     (only when review 1 had a finding the owner chose to fix)
3. review 2        (only when fix round 1 happened)
4. fix round 2     (only when review 2 had a finding the owner chose to fix)
5. review 3        (only when fix round 2 happened)
6. the final message
```

A review that found nothing to fix skips straight to step 6. Review 3 confirms the fixes; it never starts another fix round. Step 6 is the final message; there is no step 7. A dispatch that is not the next number in this list is a budget violation: write the final message instead.

Size check: when the scope is one file or under about 100 changed lines, offer one reviewer pass and no fix-round budget, and say so in the first message. Use the full budget only when the owner asks for it or the scope is larger.

Three things hold it:

1. **Say it up front.** Your first message is the block in the intro: the budget sentence, the scope, the change summary, and `review budget: reviews 0/3, fix rounds 0/2 | next: review 1`.

2. **Count it.** Every message that leads to a step ends with the counters line, with that step already counted and the next step named: `review budget: reviews 1/3, fix rounds 0/2 | next: fix round 1`. Print it before the step, never after: nothing happens before the message that carries its counters. Keep a copy where it survives a context reset: in the repo's plan workspace (`.superpowers/sdd/<plan>/review-budget.md`) when there is one, otherwise in a scratch file outside the repo.
3. **Refuse past the cap.** Check the counters before dispatching. At the cap, do not dispatch. Stop and send the owner one final message: "budget exhausted; remaining findings: <list>; recommend accepting / fixing manually / dropping".

Before every dispatch, check four things: the step number, the counters, the scope, and the earlier findings.

- **Step number**: is this dispatch the next number in the list above?
- **Counters**: is a pass left for this dispatch?
- **Scope**: is it the files your last receipt named, written as a list of paths?
- **Earlier findings**: does this report repeat a finding an earlier round already fixed? Fix it again, but the loop does not grow: the pass count does not.

A run ends in exactly one terminal state, and the final message always states the counters (reviews used, fix rounds used), the nits, and the terminal state.

Only the owner can raise the budget, ask for another round, or stop the run. You never raise it yourself. An owner request for a round of fixes counts against the budget, even when the findings are nits. An answer of "review again" never adds a step: it only tells you to keep going to the next step in the list above. The budget changes only when the owner says to raise it and names the new number.

## 1. Pick the scope

First match wins. Do not ask when one matches.

1. The user names files, commits, a branch or a PR: review that. Diff: see "Named scopes" below. If the request names a feature, function or mutation instead of files ("review the mutation"), the scope is the files that implement it: say which. If you cannot tell which files, ask one question before you dispatch. A request narrower than this session's changes is never widened to all of them.
2. This session changed files and they are still uncommitted: review those files, even on a feature branch. Diff: `git diff HEAD -- <paths>`.
3. Uncommitted changes exist, but this session did not make them: review every uncommitted change, and say so in your first message. A session with no history of its own cannot tell whose change is whose, so it takes the whole tree. Diff: `git diff HEAD`.
4. On a branch other than the default branch: review the whole branch. Diff: `git diff $(git merge-base HEAD <default>)`.
5. On the default branch with nothing uncommitted: ask the user for a branch or PR, then stop.

The default branch is `origin/HEAD` (for example `origin/main`). If it is not set, use `main` or `master`. If neither exists, ask.

**Whose change is it.** Another session may be working in the same checkout. `git status` shows its files as uncommitted too, and they are not yours to review: the reviewer would spend the round on someone else's work, and the handoff would claim it. This session knows the files it wrote and edited; that is the scope. Include a file another session changed only when the change you are reviewing breaks because of it, and say which file and why.

The Scope slot of the reviewer prompt is a list of paths, never a phrase like "the uncommitted tree" or "the files I changed". Write the paths out as a comma-separated list. For rule 3, take them from `git status --porcelain` and `git diff --stat`.

### Named scopes

- Files: `git diff $(git merge-base HEAD <default>) -- <files>`. If that is empty, put "none: review these files whole: <files>" in the Diff slot, and keep it for every round.
- Commits: `git diff <oldest commit>^`, for example `git diff HEAD~3` for the last 3 commits. If later commits or uncommitted changes exist, this diff includes them: ask the user whether to include them. If not, use `git diff <oldest commit>^ <newest commit>`.
- A branch or PR: you edit the working tree, so the branch must be checked out. If it is not, ask before you run `git switch <branch>` or `gh pr checkout <pr>`. With uncommitted changes, do not check out: ask the user to commit or stash them, then stop. Diff: rule 3. The base is the default branch, or `origin/<baseRefName>` for a PR.

For a PR, first run `gh pr view <pr> --json number,title,body,headRefName,baseRefName` and `git fetch origin <baseRefName>`.

### PR context

Whenever the scope is a branch (rule 4, or a named branch or PR), look for its PR before you write the change summary. Skip this when `gh` is missing or the scope is not a branch.

1. Find it: `gh pr list --head <branch> --state all --json number,title,state,url --limit 1`. No result means no PR: carry on, and say "no PR found" in the first message.
2. Read it, using the number from step 1:
   - `gh pr view <n> --json title,body,reviews,comments,url`
   - `gh api repos/{owner}/{repo}/pulls/<n>/comments --jq '.[] | {path, line, user: .user.login, body}'` for inline review comments.
3. Keep what carries intent: the description, requirements and acceptance criteria, decisions the author or reviewers settled, and review comments that are still open. Drop bot noise, CI status and "LGTM".
4. Use it in three places:
   - Step 2: the PR is the best source for the change summary after this session.
   - Step 3: settled decisions go into the `Settled decisions` slot, and open review comments go into `What the change should do` as things to check. Write each as one line with its source (`PR #<n>`).
   - Step 7: the PR link, the decisions and the open comments go into the handoff `Context` and `Open` lines.
5. Name the PR (number and URL) in the first message.

### Fix rounds

A fix round reviews only the files that changed in that round, never the whole tree.

The next round's scope is exactly the files your last receipt named ("1: fixed (<files>)"), and nothing else. Take the list from the receipt, never from `git status`: the original change's files are still modified in the working tree, and they are not in scope again.

Example: your receipt says "1: fixed (README.md)" and "2: fixed (test_jobs.py)". The next review's Scope is exactly `README.md, test_jobs.py`. It is not `jobs.py`, even though `jobs.py` was in the original change and is still modified in the tree.

The diff command of a fix round must include the uncommitted fixes: if it has a file filter, an end commit or no diff, also give `git diff HEAD`.

Resolve the scope once per round and treat it as fixed for that round.

## 2. Write the change summary

Write 2 to 5 lines on what the change should do: its goal and requirements. Leave out how it was built and why you think it works, because that anchors the reviewer.

This summary is the reviewer's binding intent: a finding must break it. It states:

- the acceptance criteria, as things someone can check;
- the decisions already taken, so the reviewer does not re-raise them;
- the size of the change (lines changed) and whether it is one logical change. Around 100 changed lines is a good review size, 300 is the edge, over 1000 belongs in pieces.

Sources, best first:

1. This session, if the changes were made here.
2. The PR found in "PR context": title, body and comments.
3. The messages of the commits in scope: `git log --format='%s%n%b' <base>..HEAD`. Uncommitted changes have none.

If none exist, write "Unknown. Infer it from the diff."

## 3. Dispatch the reviewer

Dispatch one fresh subagent with this prompt. Fill in the `<slots>` and copy the rest word for word.

Preflight: run the diff command with `--stat`. If it is empty, the scope holds no changes: do not dispatch. Say so and stop.

~~~text
Review a code change. Report only: do not edit files, commit, stash or switch branches. Never modify a file, not even temporarily (no inverting a condition to test it). Read, grep and run the existing tests only.

Repo: <absolute path>
Scope: <the paths in scope, comma separated, for example "lib/a.py, test/a_test.py". In a fix round this is exactly the files your last receipt named, and nothing else.>
Diff: run `<diff command>`, and read the untracked files from `git ls-files --others --exclude-standard`
What the change should do: <change summary>
Settled decisions: <the plan ledger, AGENTS.md, the plan or spec of the repo, the linter config, and decisions from the PR (see "PR context"). Write "nothing" if there are none.>
Earlier findings: <"none", or one line each: <title> (<path>): fixed | not chosen | skipped, <reason>>

Review only the files and hunks named in Scope. Everything else in the tree was
reviewed in an earlier round — do not re-review it and do not report anything
about a file outside Scope.

Intent (binding): <change summary>
A finding counts only if it breaks the Intent above, breaks a rule quoted from
AGENTS.md, or loses data. Shape, wording, comment phrasing, dead code and
"could be simpler" are NOT findings for this review.

Rules for your report:
1. Each finding must cite a file:line you actually read AND a concrete situation
   (specific input or event) that produces the wrong result — or, for dead
   weight, the concrete reason it earns nothing. No such case, no finding.
2. Drop findings about code this change did not touch, and anything an earlier
   round already reported.
3. Decisions recorded in <ledger, plan or AGENTS.md> are settled. Do not re-raise
   them.
4. At most 5 findings, ranked must-fix first. If you found more, list the top 5
   and state how many you dropped.
5. Verify before you report: for any claim of the form "X is missing", "X is
   never called", "this branch is dead", run the command that shows it (grep, a
   test, a build) and quote the output. If the claim turns out to be wrong, do not
   report it — say so under "Checked and found correct".
6. Quit condition: if you have no must-fix and no should-fix findings, reply
   exactly "Verdict: Ship." plus one line on what you checked, and stop. Nits
   alone are never a reason to keep going.
7. If the code marks something as deliberate — a `skip`, an "on purpose" comment,
   a guard with a reason next to it — do not report it as a finding. Note it once
   under "Deliberate, left alone", quoting the marker, and leave it alone.

1. Read the diff, then the code around it: the callers of each changed function, what it calls, its tests and the README. When a signature or behavior changes, find every caller.
2. Read the project rules: AGENTS.md and CLAUDE.md in the repo root and in the folders of the changed files.
3. Load no other skill. Use the repo's own rules (AGENTS.md, CLAUDE.md, the linter config) as the style checklist. A finding that only changes how the code reads is a nit, never a should-fix. Restructuring is the simplify pass's job, and the owner runs that pass.
4. Look for what breaks the Intent:
   - Bugs: wrong results, crashes, edge cases (empty, zero, last item, rounding, time zones), and callers the change breaks.
   - Risk: injection, leaked secrets, unchecked input, swallowed errors, and writes that can lose data.
   - Load: races, unbounded growth, a query per item, and slow code on big inputs. Judge against the expected load in the README or config. Name the load you assumed in each load finding.
   - Tests: risky new logic with no test that fails when it breaks.
   - Dead weight: code or tests the change added that earn nothing. A test asserts nothing or only trivially true things, asserts what the mock it set up returned, duplicates another test on the same path, or only checks that something exists. Dead code the change added that nothing calls counts here too. Name the reason it earns nothing.
   - Rules: project rules the change breaks. Quote the rule.
   Drop anything that only changes how the code reads.
5. Check each finding before you report it. Re-read the lines and name a concrete case: this input or situation gives this wrong result, or this test earns nothing because of such and such. No case, no finding. Drop:
   - problems the change did not add
   - what a linter or type checker catches
   - taste, and anything you would phrase as "consider"
   - earlier findings marked "not chosen"
   Check each earlier finding marked "fixed". Report a broken fix as a new finding.

Zero findings is a valid result. Do not invent one. Keep the report under 400 words unless a finding needs more: a security finding or an architectural objection needs a paragraph, not a line.

Severity:
- Must fix: a bug, a security hole, data loss, or a break at the expected load. Blocks the run.
- Should fix: risky logic with no test, real slowness, a broken project rule, flaky tests, dead code or a test that earns nothing that the change added, or logic an existing helper or library already does (name it). Fix these before shipping.
- Nits: findings that only change how the code reads, such as naming, formatting, comments and needless nesting. Never higher, never a blocker. Comment slop counts as a nit: a banner separator, a step label ("# Step 1:"), a comment that restates the line below it, an empty label, ALL CAPS, or a version narrative ("previously X", "now supports Y").

Reply with only this report. Use plain English, and explain a term the first time you use it. Answer in the language the request was written in.

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

Checked and found correct: <one line each, for a claim you checked and dropped>
Deliberate, left alone: <one line each: <path>:<line> and the marker that says so>
Not checked: <one line, only if something that matters could not be checked>

Number the findings across all groups. Leave out empty groups. With no findings, write "Verdict: Ship.", the scope, and one line on what you checked.
~~~

## 4. Check the findings, then ask what to fix

Print the findings in your own message text, before you ask anything. The owner does not see the subagent's output, and a question with no findings above it is unanswerable. This applies to every review, including review 2 and 3, and including "Verdict: Ship." (print the verdict and the one-line check).

- Print the full report as the reviewer wrote it, or a faithful condensed version: verdict, scope, and every finding with number, severity, title, `path:lines`, problem and fix. Never drop or soften a finding, and never replace the findings with "the reviewer found 3 issues".
- Print the nits too, as a short list.
- Only after the printed findings, ask which to fix. The question tool does not replace the printed report: it goes in the same turn, after it.

**Gate 1 — the spot-check, before the owner sees the list.** A finding that asserts something is absent, unused, dead or unreachable is the kind that turns out to be wrong. Run the check for each must-fix and should-fix finding, for that one claim and nothing else:

| Claim | Check |
|---|---|
| "X is never called / not stored / missing" | `grep -rn "<symbol>" lib/ test/` (or the equivalent in the repo) and read the hits |
| "this branch is dead / unreachable" | write a throwaway 3-line test or trace the caller by hand, then delete it |
| "the docs say Y" | open the file and quote the line |
| "this breaks the contract" | quote the contract line from the spec or AGENTS.md and name the input that breaks it |
| "this branch is wrong" but the code marks it deliberate | quote the marker (a `skip`, an "on purpose" comment, a guard with a reason next to it) and drop the finding |
| "this new logic has no test" | invert one condition the change adds, run the suite, restore the file; a mutation that stays green is the finding |

The check decides, not the owner's pick:

- The check confirms the finding: it survives gate 1, and goes on the list below.
- The check refutes it: it is dropped. Do not fix it and do not offer it to the owner, even when the owner would have picked it. Report it in the final message with the command and its output.

Read the hits, do not just count them: an empty result is not proof of absence either. A symbol that a grep does not find may live under another name, in another directory, or behind a caller.

A refuted finding is not an open finding: it does not keep the run open.

When no must-fix or should-fix finding survived, fix nothing: go to step 6 and close the run. Do not ask, and do not dispatch another review.

Then ask which of the survivors to fix. The first option is always your own recommendation: the findings you judge worth fixing, named by number, and one line on why the rest are not. Judge by what each finding's Fix line asks for: a one-line fix is worth taking, a refactor bigger than the defect is not.

```
Which findings should I fix?

- Fix 1 and 3 — 2 needs a refactor bigger than the defect; leave it.   (recommended)
- Fix every must-fix and should-fix finding
- Only must fix
- Done
```

What goes in your recommendation, by number:

- every must-fix finding that survived gate 1;
- a should-fix finding when its Fix line asks for a small change;
- never a nit, and never a should-fix whose fix is a larger change than the defect: leave it out and name the trade-off.

If you would fix everything, the first option says so and the duplicate option goes. "Done" is never the first option.

Four options at most, and the recommendation is the first. Print the list, one option per line: do not paraphrase it into a sentence. The user can also type numbers, for example "1 and 3", or name findings the options did not offer. Fix nothing before the user answers.

The answer decides what gets fixed:

- It names findings: fix exactly those, and only those.
- It is "Done": fix nothing. Go to step 6 and close the run.
- It names the nits, or asks for a round of fixes: that is a request for a round. Fix what it named, and count the round against the budget.

When the answer names nothing to fix, there is no fix round: go to step 6 and close the run. Do not ask for another review, and do not dispatch one.

Nits are never on the options list. They need no decision: the final message lists them once, unless the owner asked for them.

If the report repeats a finding that an earlier round already fixed, the loop is not converging: fix it once more and keep the shape. The review count does not grow. Compare each finding with the "Earlier findings" list before you ask the owner what to fix; that list says which findings were already fixed.

## 5. Apply the fixes

Only the findings the owner named are fixed. Everything on the list survived gate 1 in step 4, so there is nothing left to reject.

Apply them yourself, in this session. You hold the context the finding does not: why the change exists, what was tried, what must stay true. That is the reason you fix it and the reviewer does not.

- One finding at a time, with the smallest change that fixes it. Touch nothing else, and do not offer other changes.
- Read the file you are about to edit, and only that file. You are applying a finding, not reviewing the change.
- For a bug, add a test that fails without the fix.
- For dead weight the change added, delete it: a test that earns nothing, or code nothing calls. Do not rewrite it into a better test unless the finding asks for it.
- For a nit about naming, comments or formatting, follow the repo's own conventions (AGENTS.md, linter config). Do not load another skill for it. Slop to remove: a banner separator, a step label, a comment that restates the line below it, an empty label, ALL CAPS, or a version narrative. Keep a comment that carries a why, a constraint or a trap.
- If a finding is wrong or unsafe to fix, skip it and say why. If you need a decision to fix it, skip it and say what decision you need; the final message asks the owner for it.
- Do not commit, push, stash or switch branches.
- At the end, run the project's tests, and its linter if it has one.

Then write the receipt, one line per finding:

```
1: fixed (jobs.py)
2: skipped, the finding is wrong: <why>
```

Then name the files you changed, the test command and its result. The receipt is what the next round's scope is computed from, so the file names must be exact.

Report a skipped finding back to the owner in the final message. Never silently drop one.

## 6. Show the receipt, cap the loop, and close the run

Show your receipt.

Then take the next step in the list above: after fix round 1 that is review 2, after fix round 2 that is review 3. Review 3 is the last review. After it, write the final message: do not start another fix round, whatever review 3 reports. What it still reports goes to the owner as accepted trade-offs, or as an owner decision when it is a finding that survived the fix.

Ask the owner whether to review again before each of those reviews. If the owner answers "review again" but the list is over, write the final message and say the budget is spent: "review again" never adds a step.

A review that found no must-fix and no should-fix finding ends the run: go to the final message. No dispatch follows it, and no "review again" question.

Then close the run. Your final message states:

- the terminal state:
  - **Ship**: no must-fix and no should-fix findings left, the project's own checks green, and the owner's acceptance criteria met.
  - **Accepted trade-offs**: findings left open, each with the reason.
  - **Owner decision required**: a finding that survived the fix, or a budget exhaustion. A finding you dropped because a check refuted it does not count as open.
- the counters: `review budget: reviews 3/3, fix rounds 2/2`.
- the handoff path, so the owner knows where the next agent will look.
- the nits, listed once. They are not fixed, not stored in a file, and not carried into the next round.
- what you verified: the checks and commands you ran, and their results.
- every finding you dropped because a check refuted it, with the command and its output.

No dispatch happens after the final message.

## 7. Leave the handoff

The final message is for the owner. The handoff is for whoever picks the work up next: another session, another agent, tomorrow. It is written once, at the end, after the fixes and before the final message, so the final message can name its path.

This session's context dies with this session. The handoff is what survives it, so it carries what the code does not show.

Where it goes:

- The repo's plan ledger (`.superpowers/sdd/<plan>/progress.md`) when the repo has one. Append to it; never rewrite what is already there.
- No ledger? A scratch file outside the repo, for example `$TMPDIR/review-handoff.md`. Never in the workspace: it would be committed noise.

What goes in it:

- **Scope**: the paths this session changed and reviewed, and the diff or commit range the review ran against. Not everything uncommitted in the tree.
- **Left out**: the uncommitted files this session did not change, so the next session knows they exist and are not part of this review. Leave the line out when there are none.
- **Terminal state** and the counters.
- **Fixed**: each finding that was fixed, with its file.
- **Open**: each finding left open, with the reason: accepted trade-off, budget exhausted, or needs a decision.
- **Decisions**: what the owner decided during this review, one line each. They are settled. A later review must not re-raise them.
- **Context**: why the change exists, what was tried and rejected, and what the owner wants. When a PR exists: its URL, the decisions from its description and comments, and the review comments still open. The next session reads this first: it is the part the code cannot say.
- **Nits**: not fixed, not blockers.
- **Next step**: what the next agent should do first.

Keep it short. It is a handoff, not the report: reference the plan, the spec and the diff by path instead of repeating them.

## Common mistakes

- Reviewing inline because the diff is small, you already read it, or other instructions prefer fewer subagents. Dispatch the reviewer anyway.
- Judging the change yourself while the reviewer works: reading the whole diff, reading files you are not about to edit, probing the behaviour, or fixing something the reviewer did not report.
- Reviewing the whole branch when there are uncommitted changes.
- Reviewing another session's uncommitted work because `git status` listed it. The scope is what this session changed.
- Claiming the whole tree in the handoff when this session changed two files of it.
- Re-reviewing the whole tree after a fix round. The scope is the files your receipt named.
- Accepting a finding that asserts absence without running the check.
- Treating nits as blockers, so every round produces more work.
- Re-raising a decision the owner already made. Check AGENTS.md, the plan and the linter config first.
- Making "no findings at all" the stop condition instead of "no must-fix and no should-fix".
- Ranking a finding that only changes how the code reads above Nits.
- Downgrading dead weight to a nit. A test that earns nothing is dead weight: should-fix, and the fix is to delete it.
- Reporting a branch the code marks as deliberate, or fixing one after the owner picked it. Quote the marker, drop the finding, say so in the final message.
- Skipping a slop comment because it is only a nit. Report it in the nits; do not fix it in a review round.
- Dispatching the first reviewer before the message that states the budget. Scope discovery, then the message, then the dispatch: not scope discovery, dispatch, message.
- Asking what to fix without printing the findings first. The subagent's report is invisible to the owner until you write it in your own message.
- Fixing before the user picks, or starting another review without asking.
- Applying a fix that goes past the finding. The fix is the finding, not your own better version of it.
- Committing the fixes.
