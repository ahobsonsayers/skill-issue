---
name: opencode-configuration
description: Use when adding, updating, or reordering entries in a user's opencode.jsonc config for OpenCode V2 — provider/model definitions (from the live models.dev catalog) or shell permission rules. Even if the user just says "add the new model" or "fix my config."
keywords: [opencode config, opencode.jsonc, opencode v2, model definitions, shell permissions, allowed-tools]
---

# Update opencode Config

## Overview

Edit the user's opencode config at `~/.config/opencode/opencode.jsonc`. This skill targets **OpenCode V2** config shapes only — do not apply it to a V1 config or `.opencode/` layout without first checking the [V1→V2 migration guide](https://opencode.ai/v2/docs/migrate-v1).

Two kinds of change today: **provider/model definitions** (translated from the live models.dev catalog into OpenCode V2's `providers.<providerID>.models.<key>` schema) and **shell permission rules** (entries in V2's ordered top-level `permissions` array, categorized, alphabetical). Other config kinds may be added as references over time.

**Every change ends with prettier + schema validation** (see Finalizing). No exceptions — not for "a one-line permission edit", not for "I only changed a number".

## Source of Truth

- **Always use the official provider for each model family.**
- **Catalog source:** the live models.dev catalog at `https://models.dev/api.json`. V2 does **not** cache this locally — there is no `~/.cache/opencode/models.json` file to read anymore. Fetch and filter it with curl + jq, e.g.:
  ```sh
  curl -sL https://models.dev/api.json | jq '.acme'
  ```

### Find the Official Provider

1. Check the live catalog (`https://models.dev/api.json`) for a provider whose `id` or `name` matches the model's creator (e.g. `alibaba` for Qwen models).
2. Prefer providers that appear first-party (the company's own API), not resellers/aggregators like `vercel`, `openrouter`, `requesty`, etc.
3. If multiple candidates exist, pick the one with the most complete model entry (most fields populated, real pricing not zeros).
4. If still unclear, **ask the user** which provider to use. Do not guess silently.

The model key in the user's config is the short id (e.g. `acme-2.1`, not `acme/acme-2.1`), nested under that provider's `models` object — e.g. `providers.acme.models.acme-2.1`.

### V2 Provider Model Fields

The OpenCode V2 config schema for an entry under `providers.<providerID>.models.<key>` only allows these fields (per the [providers guide](https://opencode.ai/v2/docs/providers)):

`modelID`, `name`, `family`, `releaseDate`, `package`, `settings`, `headers`, `body`, `capabilities`, `compatibility`, `variants`, `cost`, `limit`, `disabled`

This is a **different shape from V1** and different from the raw models.dev source catalog. Do not confuse the two:

- The map key (e.g. `acme-2.1`) is the OpenCode-facing model id. There is no `id` field inside the object.
- `modelID` is a distinct, optional field — only set it when the provider's own model string differs from the map key (aliasing), e.g. `"modelID": "acme-2.1-20260101"` under a friendlier key like `coding-default`.
- `attachment`, `modalities`, `tool_call`, `reasoning`, `temperature`, `options`, `interleaved`, `status`, `provider`, `experimental` are **not** V2 fields. They existed in V1 and/or are raw models.dev source fields — translate them, don't copy them (see references/models.md).
- `options` (V1) is renamed `settings` (V2).
- `variants` changed from a V1 **object** keyed by variant name to a V2 **array** of `{id, settings, headers?, body?}` objects.

## Detailed References

Load the matching reference when working on that kind of change:

- **[references/models.md](references/models.md)** — model definitions: required steps, translating source catalog fields into V2 `capabilities`/`settings`/`variants`, variant tables per family, field ordering, cost block pattern, common mistakes.
- **[references/permissions.md](references/permissions.md)** — shell permission rules: the V2 `permissions` array shape, categories, alphabetical rule, config-file shadowing, common mistakes.

## Finalizing

Every model or permission change **must** end with these two steps, no exceptions:

1. Run **prettier** on the file. Do not use `// prettier-ignore`; accept prettier's natural formatting for arrays and objects. Use `prettier --parser jsonc ~/.config/opencode/opencode.jsonc` (formatter wrappers may not handle JSONC comments).
2. **Validate** the config against the opencode schema with zero errors:

```python
import json, re, subprocess, sys
from pathlib import Path

# Fetch schema (urllib gets 403; use curl)
result = subprocess.run(["curl", "-sL", "https://opencode.ai/config.json"], capture_output=True, text=True)
schema = json.loads(result.stdout)

# Load + clean config
raw = Path.home().joinpath(".config/opencode/opencode.jsonc").read_text()
clean = re.sub(r"(?<![\":])//[^\n]*", "", raw)
clean = re.sub(r",(\s*[}\]])", r"\1", clean)
cfg = json.loads(clean)

# Validate
import jsonschema
validator = jsonschema.Draft7Validator(schema)
errors = sorted(validator.iter_errors(cfg), key=lambda e: list(e.absolute_path))
if errors:
    for e in errors:
        print(f"{' → '.join(str(p) for p in e.absolute_path)}: {e.message}")
    sys.exit(1)
else:
    print("✅ Valid")
```

Schema-validating the config is mandatory because JSONC parses fine even when a permission entry is malformed or a model field is non-schema. Only `jsonschema.Draft7Validator` catches those.

The schema at `https://opencode.ai/config.json` covers the V2 top-level keys (`permissions`, `providers`, `agents`, `update`, `share`) and still lists some V1 keys, with `additionalProperties: false`. Any error it reports is a real problem, unless it can lag behind newer V2 docs: if it flags something the V2 docs in this skill and its references say is valid (e.g. `variants` as an array, a `capabilities` object), trust the docs and report the discrepancy to the user instead of changing correct config to satisfy the schema.

## Red Flags — STOP and Fix (shared)

- "Prettier is expanding the arrays, so I'll add `// prettier-ignore`." → Do not use `// prettier-ignore`. Accept prettier's formatting.
- "I'll just check it parses as JSON, that's enough." → Always validate against the opencode schema too.
- "It's just a one-line edit, no need to validate." → Run prettier + schema validation on every change.
- "The schema validator flagged this, so it must be wrong for V2." → Check whether it actually matches the documented V2 shape in this skill first; the schema may lag the V2 docs (see Finalizing).
- For model-specific or permission-specific red flags, see the respective reference file.
