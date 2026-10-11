---
name: github-workflows-style
description: Use when writing, editing, or reviewing GitHub Actions workflow files (.github/workflows/*.yml) — field ordering, permissions placement, step-id usage, GITHUB_ENV set-before-export, hyphenated IDs/contexts, unquoted values, and run: hygiene (no context expressions, for loops, arrays, or braces). Even if the user just says "add a workflow" or "fix this GHA file."
---

# GitHub Actions Workflows Style

Workflow YAML in this repo must match these conventions. Apply to all new and edited `.github/workflows/*.yml` files; flag violations when reviewing. These rules are enforced by ast-grep rules in `~/.config/ast-grep/rules/github-actions/`.

## When to Use

- Writing or editing `.github/workflows/*.yml`
- Reviewing a workflow diff or PR for style compliance
- When unsure how to order fields or name an ID/context

When NOT to use: do not apply to other YAML files (docker-compose, Helm, etc.) or third-party reusable workflows you don't own.

## Quick Reference

| Rule                | Violation                              | Compliant                              |
| ------------------- | -------------------------------------- | -------------------------------------- |
| Top-level permissions | `permissions:` on each job            | one `permissions:` at workflow top level |
| `id:` only with outputs | `id:` on a step nothing references   | `id:` only when `steps.<id>.outputs` is referenced |
| Set before export   | `echo "VERSION=1.2.3" >> $GITHUB_ENV`  | assign, then export to `$GITHUB_ENV`    |
| Job `name:` first   | `runs-on:` before `name:`              | `name:` then `runs-on:`                |
| Job `name:` present | missing `name:` field                  | every job has a `name:`                |
| Step field order    | `uses:` before `id:`                   | `name` → `id` → `uses` → `with` → `run` → `env` |
| Hyphenated IDs      | `id: build_image`, `inputs.tag_name`   | `id: build-image`, `inputs.tag-name`   |
| Unquoted values     | `runs-on: 'ubuntu-latest'`             | `runs-on: ubuntu-latest`               |
| No `${{ }}` in run  | `run: echo "${{ inputs.tag }}"`        | bind via `env:` then `echo "$TAG"`     |
| No `for` loops      | `for item in ...; do`                  | `while read -r item; do`               |
| No bash arrays      | `arr=(one two three)`                  | newline-delimited + `while read`      |
| No `${VAR}` braces  | `echo "${HOME}"`                       | `echo "$HOME"`                         |
| Heredoc output      | `echo "key=value" >> $GITHUB_OUTPUT`   | heredoc `key<<EOF ... EOF`             |

## Top-Level Permissions

Declare `permissions:` once at workflow top level — it applies to all jobs unless a job overrides it. Never repeat identical `permissions:` blocks per job:

```yaml
# good
permissions:
  contents: read
  packages: write

jobs:
  build:
    name: Build
    ...

# bad — duplicated per job
jobs:
  build:
    name: Build
    permissions:
      contents: read
      packages: write
  push:
    name: Push
    permissions:
      contents: read
      packages: write
```

Add `permissions:` inside a single job only when that job genuinely needs different (typically narrower) scopes than the rest.

## Step IDs Only When Referenced

Give a step an `id:` only if something later references it (`steps.<id>.outputs.*` or `steps.<id>.conclusion`). `id:` is not a label — `name:` already identifies the step. Most steps need no `id:`:

```yaml
# good — no id, nothing references it
- name: Build image
  run: make build

# good — id earns its place: an output is consumed
- id: get-image-slug
  run: ... # writes to $GITHUB_OUTPUT
- run: echo "${{ steps.get-image-slug.outputs.slug }}"

# bad — id with no consumer
- id: build-image
  run: make build
```

## Set Version Before Export

In a `run:` step, compute/assign a value first, then export it to `$GITHUB_ENV` (or `$GITHUB_OUTPUT`) as the final line. Keep the export step trivial — one echo:

```yaml
# good
- run: |
    VERSION="1.2.3"
    echo "VERSION=$VERSION" >> "$GITHUB_ENV"

# bad — literal inlined into export
- run: echo "VERSION=1.2.3" >> "$GITHUB_ENV"
```

## Job Field Ordering

Every job must have `name:` and it must be the first field:

```yaml
# good
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - run: make build

# bad — missing name
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: make build

# bad — name not first
jobs:
  build:
    runs-on: ubuntu-latest
    name: Build
    steps: [...]
```

## Step Field Ordering

Steps use a fixed field order. This order is mandatory; any field present must appear in this position, no exceptions:

```
name → id → uses → with → run → env
```

`name:` is always first when present. `id:` always follows `name:` and precedes `uses:`. Never put `id:` before `name:` — if the step has both, `name:` is first, full stop.

```yaml
# good
- name: Checkout
  id: checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
  run: make build
  env:
    MODE: release
```

Omit fields that don't apply, but keep the remaining ones in this exact order.

## Hyphenated IDs and Contexts

User-defined identifiers (step IDs, `inputs.*`, `outputs.*`, `vars.*`) use hyphens, not underscores. Built-in GitHub contexts (`github.*`, `env.*`, `matrix.*`, `secrets.*`) keep their native names.

```yaml
# good
- id: build-image
  run: make build
echo "${{ inputs.tag-name }}"
echo "${{ outputs.my-output }}"

# bad
- id: build_image
echo "${{ inputs.tag_name }}"
echo "${{ outputs.my_output }}"
```

## Unquoted Scalar Values

Don't quote values unless YAML requires it (strings containing special chars, numbers that must stay strings):

```yaml
# good
runs-on: ubuntu-latest
env:
  MODE: production
  VERSION: 1.0.0

# bad
runs-on: 'ubuntu-latest'
env:
  KEY: "value"
  TAG: 'latest'
```

## run: Hygiene

### No Context Expressions in run:

Never put `${{ }}` inside `run:`. Bind context values via `env:` instead:

```yaml
# good
- name: Tag
  env:
    TAG: ${{ inputs.tag-name }}
  run: echo "$TAG"

# bad
- name: Tag
  run: echo "${{ inputs.tag-name }}"
```

### No for Loops

Use `while read -r` over newline-delimited input, never `for`:

```yaml
# good
- run: |
    while read -r item; do
      echo "$item"
    done <<< "$(printf 'a\nb\nc')"

# bad
- run: |
    for item in a b c; do
      echo "$item"
    done
```

### No Bash Arrays

No `arr=(...)`. Use newline-delimited input with `while read`:

```yaml
# good
- run: |
    while read -r item; do
      echo "$item"
    done <<< "$(echo -e 'one\ntwo\nthree')"

# bad
- run: |
    items=(one two three)
    echo "${items[@]}"
```

### No ${VAR} Braces

Use `$VAR` unless a suffix/prefix makes braces genuinely needed:

```yaml
# good
- run: echo "$HOME"

# bad
- run: echo "${HOME}"
```

### Heredoc for Multi-line GITHUB_OUTPUT

For multi-line or complex outputs, use a heredoc delimiter:

```yaml
# good
- run: |
    {
      echo "result<<EOF"
      echo "line one"
      echo "line two"
      echo "EOF"
    } >> "$GITHUB_OUTPUT"

# bad — single-line for multi-line content
- run: echo "result=line one" >> $GITHUB_OUTPUT
```

## Single-line GITHUB_OUTPUT

Assign first, then a simple echo. Never inline transformations inside the `echo` that writes to `$GITHUB_OUTPUT`:

```yaml
# good — compute, then output
- id: get-image-slug
  run: |
    SLUG="build-$(echo "$IMAGE_NAME" | tr '/:.' '---')"
    echo "slug=$SLUG" >> "$GITHUB_OUTPUT"

# bad — transformation inline in the output line
- id: get-image-slug
  run: echo "slug=build-$(echo "$IMAGE_NAME" | tr '/:.' '---')" >> "$GITHUB_OUTPUT"
```

## Step IDs Are Verb-Noun

Step IDs follow `verb-noun` naming like existing steps in the repo (`checkout-code`, `get-digests`, `setup-buildx`), and every step has a `name:` phrased the same way (`Get image slug` → `id: get-image-slug`). Rename IDs and their `steps.<id>` references together:

```yaml
# good
- name: Get image slug
  id: get-image-slug
  run: |
    SLUG="build-$(echo "$IMAGE_NAME" | tr '/:.' '---')"
    echo "slug=$SLUG" >> "$GITHUB_OUTPUT"

# bad — noun-only id, no name
- id: image-slug
  run: ...
```

## Artifact Names in Reusable Workflows

Artifacts are scoped to the workflow RUN, not to a reusable-workflow call. If a reusable workflow writes artifacts that it (or its consumer jobs) reads back within the same run, and the caller may invoke it more than once per run, the artifact names MUST be namespaced per call — e.g. derive a prefix from a required input (`image-name`) in a `prepare` job and use it as the shared grouping key on both the write and read sides (e.g. `matrix-step-name` for GoCodeAlone/cloudposse matrix-outputs actions). A hardcoded name like `build` collides across calls (last-writer-wins on same matrix keys).

## Common Mistakes

- Repeating `permissions:` on every job — declare once at workflow top level; per-job only for genuinely different scopes.
- Adding `id:` to steps nothing references — `id:` is for consumed outputs only, not a label.
- Exporting literals straight to `$GITHUB_ENV` — assign first, export last.
- Putting `uses:` before `id:` — fixed order is `name` → `id` → `uses` → `with` → `run` → `env`.
- Omitting `name:` on a job — every job must have one, first.
- Using `inputs.tag_name` or `id: build_image` — hyphens, not underscores (built-in `github.*`/`secrets.*`/`env.*`/`matrix.*` are exempt).
- Quoting scalars that don't need it — `runs-on: ubuntu-latest`, not `'ubuntu-latest'`.
- Inline `${{ }}` in `run:` — bind to `env:` and reference `$VAR`.
- `for` loops or `arr=(...)` arrays in `run:` — use `while read -r` over newline-delimited input.
- `${VAR}` when `$VAR` suffices — drop braces unless needed for disambiguation.
- Single-line `echo "key=value" >> $GITHUB_OUTPUT` for multi-line values — use heredoc.