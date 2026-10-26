# skill-issue

A collection of skills I've built to make LLM coding agents work the way I want.

Skills live in this repo and are symlinked into `~/.agents/skills/` via `task`.

## Commands

- `task install` — `git pull`, then symlink skills into `~/.agents/skills/`
- `task remove` — remove skills from `~/.agents/skills/`

## Install

```bash
git clone https://github.com/ahobsonsayers/skill-issue "$HOME/skill-issue"
cd "$HOME/skill-issue"
task install
```

Run `task install` again anytime to pull latest changes and refresh symlinks.
