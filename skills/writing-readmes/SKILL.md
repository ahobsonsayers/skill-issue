---
name: writing-readmes
description: >
  Use when creating, updating, or improving a README.md for a project. Applies
  to new projects without a README, stale READMEs that need refreshing, or adding
  a section. Defaults to open-source style with a license. Keeps READMEs short,
  to the point, and easy to scan.
keywords: [readme, readme.md, documentation, license, open source, markdown]
---

# Writing READMEs

A README answers the questions your audience will have. **First ask: who will read
this, and what do they need to know?** Keep it short, scannable, and engaging. This
skill defaults to open source style — make sure a `LICENSE` file exists, but never add
a License section to the README itself (the file owns that content).

## Task Types

| Task | When |
|------|------|
| **Creating** | New project, no README yet |
| **Adding** | Need to document something new |
| **Updating** | Capabilities changed, content is stale |
| **Reviewing** | Checking if the README is still accurate |

## Project Types

Always write open source style. No need to ask the project type — this skill always
uses the open source format below.

## Process

1. **Review the project first** — read source, `package.json`/manifest, existing docs.
   Only write about what the project actually does. Don't guess.
2. **Look at well-written READMEs for tone and structure** — use a couple of polished
   open source examples as inspiration before drafting.
3. **Name + one-line description** — what it is, in one sentence. If a logo or icon
   exists, use it in the header.
4. **Intro paragraph** — 2-3 sentences on the problem it solves and why someone would
   use it over alternatives.
5. **Features** — short bullet list (with one-line explanations).
6. **Installation** — the actual command(s). Never assume setup is obvious.
7. **Usage** — one minimal working example that shows the most common case. Add more
   only if genuinely useful.
8. **Link to dedicated files** (`LICENSE`, `CHANGELOG.md`) rather than repeating their
   content — never add standalone LICENSE/CONTRIBUTING/CHANGELOG sections.
9. **Ask the user:** _"Anything else to highlight that I might have missed?"_

## Style Rules

- **Short and to the point.** Trim every sentence that doesn't earn its place.
- **No unnecessary detail.** Use headers, tables, and lists — never walls of text.
- **Engaging, not generic.** Write like a person, for a real audience.
- **Don't overuse emojis.** A few for feature bullets is fine; don't sprinkle them.
- **Show, don't just tell** — examples beat prose.
- **Use GFM** and GitHub admonitions (`> [!TIP]`, `> [!NOTE]`) where appropriate.

## Structure (open source)

```markdown
# Name

[logo/icon in header if one exists]

[badges: license, build, package version] (optional)

[one-line description]

[2-3 sentence intro: what problem, why use it]

## Features

- ...

## Installation

[command]

## Usage

[one minimal working example]

## Documentation

[link to docs or expand]
```

## Avoid

- **LICENSE / CONTRIBUTING / CHANGELOG sections** — there are dedicated files for those;
  don't add the sections. A `LICENSE` file should exist for OSS but not be a README section.
- **Overly long or complete** READMEs when the project is small — match length to scope.
- **Stale content** — when updating, remove outdated sections; if asked, note when it
  was last reviewed.
- **Assuming** the reader knows how to install or run the project.
- **Generic tone** — write for the project's real audience, not everyone.

## Checklist

- [ ] Reviewed the actual project first
- [ ] Studied a polished example README for tone/structure
- [ ] One-sentence name + description
- [ ] Logo/icon in header if one exists
- [ ] Short intro (why use it)
- [ ] Features bullets
- [ ] Install command
- [ ] One usage example
- [ ] LICENSE file present (for OSS) — no License section in the README
- [ ] No LICENSE/CONTRIBUTING/CHANGELOG sections
- [ ] Concise, scannable, no walls of text
