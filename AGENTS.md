# skill-issue

A collection of skills for LLM coding agents. Markdown-first; scripts only when a skill needs executable helpers.

## Public content policy

This is the **public** repo: every skill here must be **generic** — cross-user and cross-platform, useful to anyone. No personal info may appear in any skill.

- No personal hostnames/URLs (self-hosted gateways, personal servers), usernames, account names, database/endpoint names, or household services.
- Skills managing your own deployments (self-hosted config, admin APIs) are **not** generic.
- A skill may target a public API (e.g. a movie database) as long as every credential is an env var (Secrets Law) and nothing identifies your accounts.
- **No user preferences**: skills here have no `## User Preferences` section or other user-specific choices.
- If a skill contains, needs, or is unsure about personal info, it does not belong in this repo.
- Decide scope before creating a skill — moving a skill out later costs commits in two repos.

## ⚠️ CRITICAL RULE: Never write to `~/.agents/skills/` directly

**Skills live in this repo only.** `task install` symlinks to `~/.agents/skills/`. Creating or editing skill files under `~/.agents/skills/` gets overwritten on the next `task install`, losing your work.

- Write new skills at `skills/<name>/SKILL.md` in this repo
- Edit existing skills at `skills/<name>/SKILL.md` in this repo
- **Never** create, edit, or delete anything under `~/.agents/skills/` — that directory is managed by `task` only
- After adding/editing a skill in this repo, run `task install` to update the symlinks

## Commands

- `task install` — `git pull`, then symlink `skills/*/` → `~/.agents/skills/` (runs `remove` first). Run again anytime to refresh; all symlinks are always rebuilt.
- `task install:relative` — same, with relative symlinks
- `task remove` — remove skill symlinks from `~/.agents/skills/`
- `task clean` — remove broken symlinks in `~/.agents/skills/`
- **Bare `task` only lists tasks** — there is no default task; don't rely on it to install

## Structure

- `skills/<name>/SKILL.md` — skill definition (required)
- `skills/<name>/references/*.md` — supplementary deep dives (optional)
- `skills/<name>/scripts/*` — allowed when a skill needs executable helpers; markdown-only otherwise
- No templates or config files in skill directories
- Keep `SKILL.md` under ~500 lines / 5k tokens; move deep detail into `references/` and tell the agent **when** to read each file (progressive disclosure)

## Common structure

Most skills follow this shape (adapt to the skill, don't force it):

- Frontmatter: `name` + `description` required. `description` must state **when** to use the skill ("Use when…") and include trigger phrases — that's what drives skill activation. Extras vary (`keywords`, `tags`, `trigger`/`triggers`, `allowed-tools`) — no single convention, don't force consistency.
- `## Configuration` (env-var skills) — env-var table + preflight + Secrets Law
- `## Workflow` / `## Process` — numbered steps; prescriptive when fragile, defaults instead of menus
- `## Common Operations` (API skills) — concrete requests; `## Troubleshooting` — `| Problem | Cause | Fix |` table
- `## Common Mistakes` / gotchas — corrections the agent would get wrong without being told; add a bullet every time you correct the agent
- Validate edits by re-running the skill against a real task; record only durable, generalizable lessons

## Naming

- **Broad capability** → plain noun (e.g. `weather`, `calendar`)
- **Single action** → `action-thing` (e.g. `update-changelog`, `decompile-apk`)
- **Frontmatter `name` must match the directory name** — the symlink exposed to agents is named after the directory, not the frontmatter

## Conventions

- **Secrets Law**: secret env vars are never read, echoed, printed, or logged — use `$VAR` references only. Non-secret URLs may be read once via `printenv`, then used as literals. **Configuration tables** use `| Variable | Required | Secret | Purpose |` with a preflight check (`[ -z "${!v}" ]`).
- **Inter-skill references** use relative markdown links: `[name](../name/SKILL.md)`. **Cross-repo deps are allowed**: a skill may depend on a skill installed separately (e.g. `closedbrowser`) — note the dependency and its owner in the dependent skill; don't vendor it.
- **No tests, lint, or typecheck** — documentation repo. Validation means re-running the skill against a real task.
- **Durable lessons only**: generalizable principles, not task-specific or file-specific notes.
- **Skill quality beats convention**: structure rules are defaults, not laws — a working, verified skill outweighs deviating from convention (user directive 2026-09).
- **Write what the agent wouldn't know**: every line answers "would the agent get this wrong without this?" If not, cut it. No generic advice, no restating tool docs.
- **Prescriptive when fragile, free when flexible**: exact sequences for fragile ops ("run exactly this, don't add flags"); explain *why* for flexible steps so the agent can adapt.
- **Provide defaults, not menus**: pick a default, mention alternatives briefly; don't list equal options.
- **When in doubt, omit**: high-signal content only.

## References

- [Agent Skills docs](https://agentskills.io/home) — homepage, overview, client showcase
- [Specification](https://agentskills.io/specification) — SKILL.md frontmatter, directory structure, progressive disclosure
- [Best practices](https://agentskills.io/skill-creation/best-practices) — scoping, gotchas, templates, validation loops
- [Quickstart](https://agentskills.io/skill-creation/quickstart) — creating a skill from scratch
- [Optimizing descriptions](https://agentskills.io/skill-creation/optimizing-descriptions) — testing and improving skill trigger accuracy
- [Evaluating output quality](https://agentskills.io/skill-creation/evaluating-skills) — test cases, assertions, grading, iteration loops
- [Using scripts](https://agentskills.io/skill-creation/using-scripts) — one-off commands, self-contained scripts, agentic script design