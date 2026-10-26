# Models Reference

Add a model to the user's opencode provider config by translating its definition from the live models.dev catalog (`https://models.dev/api.json`) into OpenCode V2's `providers.<providerID>.models.<key>` schema. The source catalog and the V2 destination config use **different field names** — this is a translation step, not a copy-and-filter step.

## Required Steps

1. **Find the model** in the live catalog under the official provider for its family (see Known Official Providers table in SKILL.md, or auto-detect for unknown families):
   ```sh
   curl -sL https://models.dev/api.json | jq '.acme.models["acme-2.1"]'
   ```
   - If not found under the exact key, search that provider's models for keys containing the model name.
   - If the official provider doesn't have the model, ask the user.
2. **Translate fields into the V2 destination schema.** Only these fields are valid inside `providers.<providerID>.models.<key>`: `modelID`, `name`, `family`, `releaseDate`, `package`, `settings`, `headers`, `body`, `capabilities`, `compatibility`, `variants`, `cost`, `limit`, `disabled`.
   - Build `capabilities` from the source's `tool_call`, `modalities`, and `attachment` fields:
     ```jsonc
     "capabilities": {
       "tools": true,        // from source tool_call
       "input": ["text"],    // from source modalities.input (add a media type here, e.g. "image" or "pdf", only when attachment/modalities confirm support — check official docs if unsure)
       "output": ["text"],   // from source modalities.output
     },
     ```
   - Use `settings` (not `options`) for request-shaping fields like `reasoningEffort` or `thinking`.
   - **Discard** these source-only fields entirely: `id`, `description`, `doc`, `last_updated`, `open_weights`, `structured_output`, `temperature`, `canonical_model_id`. They have no V2 destination field. (`release_date` maps to the optional `releaseDate`.)
   - `reasoning` and `reasoning_options` are source-only signals used to *decide* whether to build `variants` (step 3) — do not copy them as fields themselves.
   - **Do NOT include `interleaved`.** It is not a V2 field. Omit it even if present in the source.
   - The map key (e.g. `acme-2.1`) is the id — there is no `id` field inside the object. Only add `modelID` when the provider's real model string differs from the map key (aliasing).
3. **Build `variants` as an array, not an object.** V2 changed `variants` from a V1 object keyed by variant name to an array of `{id, settings, headers?, body?}` objects. Build the full set from official provider docs, not just the source catalog's `reasoning_options` (which is often incomplete):
   - **Effort levels supported** (e.g. `reasoningEffort` of `high`, `medium`, `low`): one variant per level the provider really exposes, plus an `off` variant if thinking can be disabled:
     ```jsonc
     "variants": [
       { "id": "high", "settings": { "reasoningEffort": "high" } },
       { "id": "low", "settings": { "reasoningEffort": "low" } },
       { "id": "off", "settings": { "thinking": { "type": "disabled" } } },
     ]
     ```
   - **Toggle only** (thinking on or off, no effort levels): create `on` and `off` variants:
     ```jsonc
     "variants": [
       { "id": "on", "settings": { "thinking": { "type": "enabled" } } },
       { "id": "off", "settings": { "thinking": { "type": "disabled" } } },
     ]
     ```
   - **Always-on or no configurable thinking**: do not add `variants`.
   - Expose only levels the provider treats as distinct. If the docs say two levels behave the same, list the real one only.
   - For unknown models, check the official docs before deciding. If no thinking control exists, omit `variants`.
4. **Set the config model key** to the short id (e.g. `acme-2.1`, not `acme/acme-2.1`), nested under `providers.<providerID>.models`.
5. **Order the fields** with all scalar/boolean values first, then multi-line objects/arrays in this order, and `cost` always last:
   - Scalars: `name`, `family`, `modelID` (only if aliasing), `disabled` (only if set)
   - Objects/arrays: `capabilities`, `limit`, `variants`, `cost`
6. **Insert in descending version order within its family** (e.g. `acme-2.2`, `acme-2.1`, `acme-2`).

7. **Keep family organization**: one blank line between families, preceded by `// FamilyName`.
8. **Add `// From <provider>` immediately inside the `cost` block**, on the line after the opening brace. Use the actual provider name (e.g. `// From acme`).

## Cost Block Pattern

V2 nests cache pricing under a `cache` object instead of flat `cache_read`/`cache_write` fields:

```jsonc
"cost": {
  // From acme
  "input": 1.4,
  "output": 4.4,
  "cache": {
    "read": 0.26,
    "write": 0,
  },
},
```

The comment reflects the actual provider the data came from. Include `cache.write` only if the source's `cache_write` field exists and is non-null.

## Common Mistakes

| Mistake                                                                                            | Fix                                                                        |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Pulling from a reseller instead of official provider                                               | Use the official provider for each family (see SKILL.md).                  |
| Writing `// From vercel` or `// From models.json`                                                  | Comment must match the actual source provider (e.g. `// From acme`).        |
| Trying to read `~/.cache/opencode/models.json`                                                     | That file doesn't exist in V2. Fetch `https://models.dev/api.json` live.   |
| Keeping `id` inside the model object                                                                | The JSON object key is the id; there is no `id` field. Use `modelID` only for aliasing. |
| Copying `attachment`, `modalities`, `tool_call` as top-level fields                                 | V2 groups these into a single `capabilities: {tools, input, output}` object. |
| Using `options` instead of `settings`                                                              | V2 renamed the request-shaping field from `options` to `settings`.         |
| Writing `variants` as an object keyed by name (`{"max": {...}}`)                                    | V2 requires an array of `{id, settings}` objects.                          |
| Writing flat `cache_read`/`cache_write` in `cost`                                                   | V2 nests them: `cost.cache.read` / `cost.cache.write`.                     |
| Including `last_updated`, `open_weights`, `reasoning_options`, `structured_output`, `temperature`, `canonical_model_id`, `description`, `doc` | None of these are V2 destination fields. Discard them after use.          |
| Keeping `interleaved` in the model object                                                            | Not a V2 field. Omit it even when present in the source.                  |
| Dropping `cost.cache.read`/`write` entirely                                                          | Cache pricing IS allowed — copy it from the source if present, just nested. |
| Inserting out of order                                                                             | Maintain descending version numbers within each family.                    |
| Breaking family comments / blank-line separators                                                   | Preserve `// FamilyName` headers and blank lines between families.         |
| Guessing the provider for an unknown family                                                        | Auto-detect by matching the model's creator, or ask the user.              |

## Red Flags — STOP and Fix

- "I'll use a reseller because it's convenient." → Use the official provider for each family.
- "I'll match the existing config's style." → Existing config may be outdated. Use the official provider.
- "The source has `structured_output`, so I'll include it." → Not a V2 field. Discard it.
- "The source has `interleaved`, and thinking mode returns `reasoning_content`, so I need it." → No. Omit `interleaved` regardless; it's not a V2 field.
- "I'll put `reasoningEffort` under `options` like the old docs said." → V2 renamed it to `settings`.
- "I'll write `variants` as `{"max": {...}}` since that's simpler." → V2 requires an array of `{id, settings}` objects.
- "The model isn't under the official provider, so I'll use another." → Ask the user.
- "I'll add it right after the first model I see." → Insert in descending version order.
- "I can't find `~/.cache/opencode/models.json`, I'll just guess the fields." → Fetch `https://models.dev/api.json` live instead; it's the real source in V2.
