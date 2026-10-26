---
name: skill-env-config
description: Use when authoring or editing a skill that reads configuration from environment variables — declaring required vs optional vars, marking secrets, writing the preflight check, and enforcing the Secrets Law that secret values are never read or echoed. Even if the user doesn't explicitly mention env vars or config.
keywords: [environment variables, env vars, config, secrets, preflight, skill authoring]
---

# Environment Variable Configuration for Skills

A reusable "Configuration" section that skills paste in to declare env vars, run a
preflight check on required vars, and enforce that secret values are never exposed.

## When to Use

- Authoring/editing a skill that needs env-var config (URLs, API keys, tokens, profiles).
- NOT for skills with no env-var config, or file-based config (`.env`, `data/auth.json`).

## How to Apply

1. Classify each var: **required** (no default, must be set) vs **optional** (default or feature-gated). Mark **secret** on any credential, token, password, or API key.
2. Paste the template into the skill body, replacing `JOBS_*`.
3. The preflight is the skill's first step — verifies required vars by presence only.
4. In every code example: secret vars appear ONLY as `$VAR`; non-secret vars may use the literal value.

## Template (paste into the consuming skill)

````markdown
## Configuration

This skill reads configuration from environment variables. **No `.env` file** — values
must live in the runtime environment.

### Environment variables

| Variable | Required | Secret | Purpose |
|----------|----------|--------|---------|
| `JOBS_URL` | Yes | No | Base URL (e.g. `http://jobs:8080`). Read once via `printenv JOBS_URL`, then use the literal value. |
| `JOBS_API_KEY` | Yes | Yes | API key for authenticating to the jobs service. **Never read, echo, print, or log.** Use `$JOBS_API_KEY` only. |
| `JOBS_PROFILE` | No | No | Default profile name. Unset = no persistence. |

### Preflight check (required vars)

Run this FIRST. It checks presence only — never prints values.

```bash
missing=0
for v in JOBS_URL JOBS_API_KEY; do
  if [ -z "${!v}" ]; then echo "MISSING: $v"; missing=1; fi
done
[ $missing -eq 0 ] && echo "All required env vars set" || echo "Set missing vars and retry"
```

If any var is MISSING: **stop and ask the user to set it.** Do not proceed.

### The Secrets Law (absolute)

For any variable marked **Secret**: **NEVER read, echo, print, log, or otherwise expose the value.** Use only the `$VAR` placeholder.

- ✅ `curl -H "X-Api-Key: $JOBS_API_KEY" "$JOBS_URL/api"`
- ❌ `printenv JOBS_API_KEY`
- ❌ `echo "$JOBS_API_KEY"`
- ❌ `env | grep JOBS_API_KEY`
- ❌ `curl -H "X-Api-Key: abc123actualkey" ...`
- ❌ writing the value into a script, env file, or doc

**No exceptions:**
- Not "to verify it's set" — use the preflight (`[ -z "${!v}" ]`), which tests presence only.
- Not "to debug a 401" — check presence via `[ -n "$JOBS_API_KEY" ]`, not by printing it.
- Not "the command needs the literal" — the shell expands `$VAR`.
- Not "the user already shared it" — still don't echo it back.
- Not "just this once".

**Red flags — STOP if you're about to run any of these:**
- `printenv <SECRET>` · `echo "$<SECRET>"` · `env | grep <SECRET>` · `cat` on a file holding the secret

**All of these mean: don't. Use `$VAR` and let the shell expand it.**

### Non-secret values

For variables marked **not secret**: read once (`printenv JOBS_URL`), then use the literal value in subsequent commands for readability. Optional vars with a default: if unset, use the documented default.
````

## Quick Reference

| Concern | Rule |
|---------|------|
| Required var | No default; preflight verifies presence; skill stops if missing. |
| Optional var | Has a default or is feature-gated; not in preflight. |
| Secret var | `$VAR` placeholder only, always. Never read/echo/print/log. |
| Non-secret var | Read once via `printenv`, use literal value afterward. |
| `.env` files | Not used. Values live in the runtime environment. |

## Classification Guide

**Secret:** API keys, bearer tokens, passwords, client secrets, long-lived auth tokens — anything the user would not want in a log or transcript.

**Not secret:** URLs, hostnames, ports, profile names, timeouts, boolean flags, server-issued session/CSRF tokens (rotate per session, not a credential).

When unsure, default to **secret** — over-marking costs readability; under-marking leaks a credential.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `printenv SECRET` in the preflight | Use `[ -z "${!v}" ]` — tests presence without printing. |
| Forgetting the preflight | Without it the skill fails deep in a workflow with a confusing 401 instead of a clear "MISSING: JOBS_API_KEY" at the top. |
| Optional var in the preflight loop | Only required vars belong there. |
| Marking a CSRF/session token as secret | Server-issued rotating tokens aren't credentials. Mark non-secret. |
| Marking everything secret "to be safe" | Over-marking hurts readability — reserve for real credentials. |