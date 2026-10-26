# skill-issue

A **public** collection of skills for LLM coding agents. Markdown-first; scripts only when a skill needs executable helpers. No build, tests, lint, or CI: validate a skill by re-running it against a real task, or with dry-run subagents in a scratch git repo.

## Public content policy

Everything pushed is public at once. A leak means a history rewrite, a force-push, and a GitHub Support purge, so check before you commit.

- **No personal info**: no hostnames/URLs, usernames, account names, database/endpoint names, household services, or your own deployments. If a skill contains, needs, or is unsure about personal info, it does not belong here.
- **Public APIs are fine** if every credential is an env var and nothing identifies your accounts.
- **Examples are fictional, generic, and jobs-only.** Use one made-up project, a job-submission API: `Job`, `Queue`, `Client`, `submit_job`, `get_job`, `cancel_job`, `job_id`, `priority`, `payload`, `timeout`, `JOB-001`, env prefix `JOBS_`. Never use names from real work, and never mix in other domains (invoices, projects, reports, images).
  - An example shows the rule and nothing else. Never copy code the user pastes or mimic its names or shape: rewrite it as a different jobs example. Cover other forms of the rule in prose.
  - When you touch a skill, rewrite any of its examples that look lifted from one real project.
  - Exceptions: a skill about a public tool or API uses that tool's own sample values (`omdb`). A skill whose subject is not application code (tool config, prose) uses neutral placeholders (`acme`, `example.com`).
- **Taste is fine, identity is not**: opinionated style rules are welcome, written as generic rules anyone could adopt. No `## User Preferences` section, no wrapper CLIs, private file layouts, or setups that only exist on the author's machine. Style rules yield to a project's own config.
- **Before finishing a skill**, scan it for real names, personal setups, and non-generic examples. Fix any hit.
- Decide scope before creating a skill. Moving one out later costs commits in two repos.

## Never write to `~/.agents/skills/`

It holds symlinks to `skills/*/`, managed by `task`. Edits there are overwritten on the next `task install`.

- New or edited skills go in `skills/<name>/SKILL.md`. Changes show up through the symlink at once.
- Run `task install` only after adding or removing a skill directory.

## Commands (go-task, `Taskfile.yaml`)

- `task install`: runs `remove`, then `git pull`, then symlinks `skills/*/`. `git pull` fails or merges on a dirty or diverged tree.
- `task install:relative`: uses `ln -sfr`, which does **not** work with macOS `ln`. Use `task install` on macOS.
- `task remove`: removes only the symlinks for skills present in `./skills/`.
- `task clean`: removes broken symlinks.
- Bare `task` only lists tasks. There is no default task.

## Structure

- `skills/<name>/SKILL.md` is required. Frontmatter `name` must equal the directory name, because the symlink is named after the directory.
- `skills/<name>/references/*.md` holds deep dives. Say in `SKILL.md` **when** to read each one (`code-style` maps file types to `references/<language>.md`).
- `skills/<name>/scripts/*` only when a skill needs executable helpers.
- No templates or config files inside skill directories.
- Keep `SKILL.md` under ~500 lines / 5k tokens.

## Writing skills

- `description` states **when** to use the skill ("Use when…") with trigger phrases. That drives activation. Extra frontmatter keys (`keywords`, `tags`, `trigger`) vary by skill; do not force consistency.
- Every line answers "would the agent get this wrong without it?" If not, cut it. No generic advice, no restating tool docs.
- Prescriptive when fragile (exact commands, no extra flags), explained when flexible. Give a default, not a menu.
- Add a `Common Mistakes` bullet each time you correct the agent. Record only durable, generalizable lessons.
- A working, verified skill beats convention.
- Env-var skills: declare vars in a `| Variable | Required | Secret | Purpose |` table, run a presence-only preflight (`[ -z "${!v}" ]`), and never read, echo, print, or log secret values (use `$VAR` only). See [skill-env-config](skills/skill-env-config/SKILL.md).
- Link other skills with relative links: `[name](../name/SKILL.md)`. For a skill installed separately, note the dependency and its owner in the dependent skill. Do not vendor it.

Spec: <https://agentskills.io/specification>. Best practices: <https://agentskills.io/skill-creation/best-practices>.
