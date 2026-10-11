---
name: go-code-style
description: Use when writing, editing, or reviewing Go code (golang) in this project, or when flagged for style violations, naming, variable names, inline error handling, comments, stdlib/library preference (samber/lo), or minimalism.
---

# Go Code Style

Go code in this project must match these conventions. Apply to all new and edited code; flag violations when reviewing.

## When to Use

- Writing or editing `.go` source or test files
- Reviewing a diff or PR for style compliance
- When unsure how to name a variable or structure an error check

When NOT to use: do not apply these preferences to third-party or vendored Go code, or when the user explicitly requests a different style.

## Quick Reference

| Rule              | Violation                               | Compliant                                 |
| ----------------- | --------------------------------------- | ----------------------------------------- |
| Descriptive names | `c := Config{}`, `r, err := http.Get()` | `cfg := Config{}`, `response, err := ...` |
| Short receivers    | `func (config *Config)`              | `func (c *Config)`                        |
| No inline assignment | `if x := f(); x != ""`               | `x := f()` then `if x != ""`              |
| Error-first       | `if ok {...} else { return err }`       | `if err != nil { return ... }` then happy path |
| Wrap errors `%w`  | `return err`, `fmt.Errorf("...: %v", err)` | `fmt.Errorf("context: %w", err)`        |
| No nested calls    | `outer(inner(...))`                     | assign `inner` to var, then `outer(var)`  |
| Minimal comments  | verbose / changelog comments            | brief, only where non-obvious             |
| Prefer stdlib / libs | hand-rolled splitter / filter loop    | `strings.Split`, `lo.Filter`                |
| Values over pointers | return `*Config` / `*Balance`       | return `Config` / `Balance`               |
| Section whitespace   | run-on steps, no blank lines        | blank line between fetch / JSON / render  |
| Split long calls     | one giant single-line call          | one argument per line                     |
| No nested calls      | `f(g(x))` chained                   | `v := g(x)` then `f(v)`                   |
| Minimal code         | speculative abstraction             | smallest working change                   |

## Descriptive Variable Names

Never use single- or two-letter variable names (`c`, `r`, `q`, `req`) for locals. Use names that describe the value: `tokenFileBytes`, `authData`, `authJSON`, `request`, `response`, `client`, `balance`, `routes`, `query`.

Exception: method receivers use a short name — one letter (or two/three only when one letter clashes). Use the first letter of the main noun in the type name (e.g. `*Config` → `c`, `*Client` → `c`, `*Balance` → `b`). Receivers are not locals; they repeat on every method of the type, so brevity is idiomatic.

```go
// bad
func (config *Config) Load() error {
```

```go
// good
func (c *Config) Load() error {
```

```go
// bad
c := Config{ClientID: "x"}

// good
cfg := Config{ClientID: "x"}
```

## No Inline Assignment

Never assign a variable inside an `if`/`switch` condition, whether for errors or any other value. Always assign first, then check:

```go
// bad
err := json.Marshal(data)
if err != nil {
	return err
}

// good
err := json.Marshal(data)
if err != nil {
	return fmt.Errorf("marshal: %w", err)
}

// good
clientID := cmd.String("client-id")
if clientID != "" {
	client.ClientID = clientID
}
```

Exception: the Go map-lookup idiom `if x, ok := m[k]; ok` is allowed; it is a built-in comma-ok form, not an assignment.

## Error-First, Happy Path Down the Left Margin

Handle errors immediately and let the happy path flow down the left margin. Never put the happy path inside an `if/else` branch with the error in `else`:

```go
// bad
if ok {
	process(result)
} else {
	return fmt.Errorf("compute: %w", err)
}

// good
if err != nil {
	return fmt.Errorf("compute: %w", err)
}
process(result)
```

## Wrap Errors with %w

Never `return err` bare; never format errors with `%v`. Always wrap with `fmt.Errorf` and the `%w` verb to preserve the error chain:

```go
// bad
return err
return fmt.Errorf("compute failed: %v", err)

// good
return fmt.Errorf("compute: %w", err)
```

## No Nested Function Calls

Assign intermediate results to variables. Don't nest one call inside another (the only exception is `fmt.Errorf`):

```go
// bad
result := strings.ToUpper(strings.TrimSpace(input))

// good
trimmed := strings.TrimSpace(input)
result := strings.ToUpper(trimmed)
```

## Whitespace Separates Sections

Use a blank line to separate the distinct logical steps of a function, matching the existing action functions (`balanceAction`, `loginAction`). A function that builds a client, fetches data, and renders should read as clearly separated blocks:

```go
func balanceAction(ctx context.Context, cmd *cli.Command) error {
	client, err := getAviosClient(ctx)
	if err != nil {
		return fmt.Errorf("get client: %w", err)
	}

	balance, err := client.Balance(ctx)
	if err != nil {
		return fmt.Errorf("get balance: %w", err)
	}

	if cmd.Bool("json") {
		return printJSON(balance)
	}

	// render...
	return nil
}
```

Reserve blank lines for top-level steps (build client / fetch / JSON-check / render), not for individual statements inside a single block.

## Split Long Calls Across Lines

Split multi-argument function calls over multiple lines, one argument per line, when the call doesn't fit comfortably on a single line or reads more clearly broken up. Put each argument on its own line and close with a trailing-comma style:

```go
// good
routes, err := client.Routes(
	ctx,
	cmd.String("origin"),
	cmd.Int("adults"),
	cmd.Bool("one-way"),
)

// bad (too long / hard to scan)
routes, err := client.Routes(ctx, cmd.String("origin"), cmd.Int("adults"), cmd.Bool("one-way"))
```

Apply the same to library calls that take a closure, e.g. `lo.UniqBy`:

```go
// good
destinations = lo.UniqBy(
	destinations,
	func(destination gavios.Destination) string {
		return destination.DestinationAirportCode
	},
)
```

When an argument is a composite literal, keep it nested inside the call's parentheses, one field per line:

```go
// good
results = append(
	results,
	searchResult{
		destination:  destination,
		availability: availability,
	},
)
```

## No Nested Function Calls

Do not nest function calls (`f(g(x))`). Assign each call's result to a variable first, then pass it to the next call. Chained/nested calls obscure data flow and make errors harder to inspect.

```go
// bad
return printJSON(client.Balance(ctx))

// good
balance, err := client.Balance(ctx)
if err != nil {
	return err
}
return printJSON(balance)
```

```go
// bad
routes, err := client.Routes(ctx, strings.ToUpper(cmd.String("origin")))

// good
origin := strings.ToUpper(cmd.String("origin"))
routes, err := client.Routes(ctx, origin)
```

Exception: a single wrapping call that is itself the operation, e.g. `return json.Marshal(data)` or `len(s)`, is fine — the rule targets chaining where an intermediate result is worth naming.

## Keep Comments Minimal

Do not remove existing comments. When editing nearby code, leave comments that are already there. For comments you write, keep them to one line max and only when the code genuinely needs clarification. Prefer rewriting the code to be self-explanatory over adding a comment.

## Prefer Libraries Over Hand-Rolled Code

Prefer a well-established library over writing your own helper whenever one already covers the need. Priority: standard library first, then a trusted third-party library. Only hand-roll when neither provides it.

- Use stdlib when it covers the need (e.g. `strings.Split` over a hand-rolled splitter, `uuid.NewString()` over a hand-decoded UUID).
- Use `github.com/samber/lo` for common slice/map helpers (e.g. `lo.Filter`, `lo.UniqBy`, `lo.FlatMap`, `lo.FilterMap`) instead of a hand-rolled loop.
- Do not reimplement what a dependency already provides.

## Values Over Pointers

Return plain values (not pointers) for data structures and config-like types. They are data, so values are simpler to work with and avoid nil handling.

```go
// bad
func (c *Client) Balance() (*Balance, error) {
	return nil, fmt.Errorf("balance: %w", err)
}

// good
func (c *Client) Balance() (Balance, error) {
	return Balance{}, fmt.Errorf("balance: %w", err)
}
```

Use a pointer only when you genuinely need to mutate or share the same instance.

## Keep It Minimal

No speculative features, no abstractions that weren't requested, no boilerplate. Write the minimum code that solves the problem. Match existing project patterns.

## Common Mistakes

- Shortening variable names to save typing — names must describe meaning.
- Assigning inside an `if`/`switch` condition (`if x := ...; ...`) — assign first, then check.
- Putting the happy path in an `if` branch and the error in `else` — check the error first, happy path flows down the left margin.
- Returning `err` bare or wrapping with `%v` — always `fmt.Errorf("context: %w", err)`.
- Nesting function calls (`outer(inner(...))`) — assign intermediates to named variables first (exception: `fmt.Errorf`).
- Hand-rolling a slice/map helper (filter, dedup, flatten) when `lo` already provides it — prefer `lo.Filter`, `lo.UniqBy`, `lo.FlatMap`, `lo.FilterMap`.
- Adding a dependency that duplicates stdlib (e.g. using `lo` when `strings`/`slices`/`maps` already covers it).
- Returning pointers for data structures and configs when a value would do.
- Running logical steps together with no blank-line separation — use a blank line between distinct blocks.
- Squeezing a multi-argument call onto one long line — split it one argument per line when it doesn't read clearly inline.
- Nesting calls inside calls (`f(g(x))`) — assign each result to a variable, then pass it on.
- Adding explanatory comments instead of making the code self-explanatory.
