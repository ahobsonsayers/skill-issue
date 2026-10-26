---
name: code-style
description: Use when writing, editing, or reviewing code in any language, or checking style, naming, formatting, error handling, nesting, or structure. Per-language references exist for Python, Go, and GitHub Actions workflows.
---

# Code Style

For Python, Go, or GitHub Actions, read the matching reference below. For any other language, skip the table and apply the rules on this page (no new reference files — add one only when a language keeps needing corrections).

| Code | Read |
| ---- | ---- |
| Python (`.py`) | [references/python.md](references/python.md) |
| Go (`.go`) | [references/go.md](references/go.md) |
| GitHub Actions (`.github/workflows/*.yml`, `*.yaml`) | [references/github-actions.md](references/github-actions.md) |

A project's own linter config or `AGENTS.md` wins on conflict. Leave generated and third-party code alone.

## All languages

Readability and simplicity over concision. Prefer more lines that are easy to read over fewer lines that are dense.

- **Assign, then use.** Put a computed value in a named variable first, then pass the variable. Do not build it inline inside a call or chain calls. This is a strong default, not a hard rule: skip it when the value is already a plain name or literal.

```text
# Do
timeout = estimate_timeout(payload)
job = submit_job(queue, timeout)

# Don't
job = submit_job(queue, estimate_timeout(payload))
```

## Never Nest — Return Early

The core rule, any language. Don't bury logic inside nested guards. Check the fast path or error first, return immediately, and keep the happy path flat at the outermost indentation. When a guard would wrap another guard, flatten it into sequential checks with early returns.

```go
// bad — nesting inside the guard
if job.Failed {
	err := retryJob(job)
	if err != nil {
		return fmt.Errorf("retry job: %w", err)
	}
}

// good — early return, flat happy path
if !job.Failed {
	return nil
}

err := retryJob(job)
if err != nil {
	return fmt.Errorf("retry job: %w", err)
}
```

When a per-language reference exists, it carries the detailed wording for that language on top of this rule.
