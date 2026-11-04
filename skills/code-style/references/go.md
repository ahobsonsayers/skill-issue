# Go

Go code must match these conventions. Apply to all new and edited code; flag violations when reviewing.

Do not apply to third-party or vendored Go code, or when the user explicitly requests a different style.

Contents: names, constructors, inline assignment, error handling (`%w`, error-first, sentinels), whitespace, long calls, nested calls, enums, interfaces, comments, libraries, values vs pointers, minimalism, common mistakes.

## Descriptive Variable Names

Never use single- or two-letter variable names (`c`, `r`, `q`) for locals. Use names that describe the value: `jobBytes`, `queueData`, `jobJSON`, `request`, `response`, `client`, `report`, `jobs`, `query`.

Exception: method receivers use a short name — one letter (or two/three only when one letter clashes). Use the first letter of the main noun in the type name (e.g. `*Queue` → `q`, `*Job` → `j`, `*Client` → `c`). Receivers are not locals; they repeat on every method of the type, so brevity is idiomatic.

```go
// bad
func (job *Job) Validate() error {
```

```go
// good
func (j *Job) Validate() error {
```

```go
// bad
i := Job{Queue: "default"}

// good
job := Job{Queue: "default"}
```

MixedCaps only — never underscores or ALL_CAPS: `maxRetries`, not `max_retries` or `MAX_RETRIES`. Acronyms are all caps or all lower: `URL`, `httpClient` — never `Url`, `Http`.

## No Stuttering, No Get Prefix

The package name is always present at the call site, so identifiers must not repeat it: `queue.New()`, not `queue.NewQueue()`; `http.Client`, not `http.HTTPClient`. The constructor is `New()` when the package has one primary type; `NewTypeName` only when it has several (`http.NewRequest`, `http.NewServeMux`).

Getters drop the `Get` prefix: `StatusName()`, not `GetStatusName()`. Keep `Is`/`Has`/`Can` on boolean predicates: `IsHealthy() bool`, not `Healthy() bool`.

## No Inline Assignment

Never assign a variable inside an `if`/`switch` condition, whether for errors or any other value. Always assign first, then check:

```go
// bad
if err := json.Unmarshal(data, &job); err != nil {
	return fmt.Errorf("unmarshal: %w", err)
}

// good
err := json.Unmarshal(data, &job)
if err != nil {
	return fmt.Errorf("unmarshal: %w", err)
}

// good
queue := cmd.String("queue")
if queue != "" {
	job.Queue = queue
}
```

Exception: the Go map-lookup idiom `if x, ok := m[k]; ok` is allowed; it is a built-in comma-ok form, not an assignment.

## Error-First, Happy Path Down the Left Margin

Handle errors immediately and let the happy path flow down the left margin. Never put the happy path inside an `if/else` branch with the error in `else`:

```go
// bad
timeout, err := computeTimeout(job)
if err == nil {
	run(job, timeout)
} else {
	return fmt.Errorf("compute timeout: %w", err)
}

// good
timeout, err := computeTimeout(job)
if err != nil {
	return fmt.Errorf("compute timeout: %w", err)
}
run(job, timeout)
```

## Wrap Errors with %w

Never `return err` bare; never format errors with `%v`. Always wrap with `fmt.Errorf` and the `%w` verb to preserve the error chain:

```go
// bad
return err
return fmt.Errorf("compute timeout failed: %v", err)

// good
return fmt.Errorf("compute timeout: %w", err)
```

Error strings are fully lowercase, no punctuation: `"invalid job id"`, not `"Invalid job ID."` Sentinel errors use the `Err` prefix and carry the package name so the origin is visible wherever the message surfaces: `ErrNotFound = errors.New("jobstore: not found")`.

## Whitespace Separates Sections

Use a blank line to separate the distinct logical steps of a function. A function that builds a client, fetches data, and renders should read as clearly separated blocks:

```go
func showJobAction(ctx context.Context, cmd *Command) error {
	client, err := newClient(ctx)
	if err != nil {
		return fmt.Errorf("get client: %w", err)
	}

	job, err := client.Job(ctx, cmd.String("id"))
	if err != nil {
		return fmt.Errorf("get job: %w", err)
	}

	if cmd.Bool("json") {
		return printJSON(job)
	}

	// render...
	return nil
}
```

Reserve blank lines for top-level steps (build client / fetch / JSON-check / render), not for individual statements inside a single block.

## Split Long Calls Across Lines

Split a call over multiple lines, one argument per line, only when it has 4+ arguments or does not fit on a single line. Two- and three-argument calls stay inline — one-per-line splitting of short calls is noise, not clarity. `gofmt` does not do this for you; split by hand, each argument on its own line with a trailing comma:

```go
// good
jobs, err := client.List(
	ctx,
	cmd.String("queue"),
	cmd.Int("limit"),
	cmd.Bool("failed"),
)

// bad (too long / hard to scan)
jobs, err := client.List(ctx, cmd.String("queue"), cmd.Int("limit"), cmd.Bool("failed"))
```

Apply the same to library calls that take a closure, e.g. `lo.UniqBy`:

```go
// good
jobs = lo.UniqBy(
	jobs,
	func(job Job) string {
		return job.ID
	},
)
```

When an argument is a composite literal, keep it nested inside the call's parentheses, one field per line:

```go
// good
attempts = append(
	attempts,
	Attempt{
		number: number,
		status: status,
	},
)
```

## No Nested Function Calls

Do not nest function calls (`f(g(x))`). Assign each call's result to a variable first, then pass it to the next call. Chained/nested calls obscure data flow and make errors harder to inspect.

```go
// bad
return printJSON(client.Job(ctx, id))

// good
job, err := client.Job(ctx, id)
if err != nil {
	return fmt.Errorf("get job: %w", err)
}
return printJSON(job)
```

```go
// bad
jobs, err := client.List(ctx, strings.ToLower(cmd.String("queue")))

// good
queue := strings.ToLower(cmd.String("queue"))
jobs, err := client.List(ctx, queue)
```

Exception: `fmt.Errorf("...: %w", err)` and a single call that is itself the operation, e.g. `return json.Marshal(data)` or `len(s)`, are fine — the rule targets chaining where an intermediate result is worth naming.

## Enums: Zero Value Is Unknown

Start iota constants with an explicit `Unknown`/`Invalid` sentinel so the zero value is not a real state:

```go
type Status int

const (
	StatusUnknown Status = iota // zero value: unset
	StatusActive
	StatusInactive
)
```

A `var s Status` silently reads as 0 — if that maps to a real state like `StatusActive`, code behaves as if a status was deliberately chosen when it was never set.

## Interfaces

- Do not define an interface next to a type with only one implementation. Start with the concrete struct; extract the interface when a second consumer, implementation, or test mock appears.
- Define interfaces in the package that consumes them, not the package that implements them. The consumer owns the contract and needs only the methods it calls.
- Accept interfaces in constructors; return concrete structs — an interface return hides every other method of the concrete type from callers.
- When implementing one, add the compile-time check near the type: `var _ JobStore = (*JobStore)(nil)`.

## Keep Comments Minimal

All comments follow the Comments section of the skill: remove slop, today's fact not history, one line unless the second carries a new fact. When editing nearby code, leave comments that are already there (unless running [simplify-code](../../simplify-code/SKILL.md), which strips narrating comments). For comments you write, prefer rewriting the code to be self-explanatory over adding a comment.

No package or file preamble comments: code should document itself. Design and usage context belongs in a README or the docs of the specific exported symbol, not in a blurb above `package` or the imports.

## Prefer Libraries Over Hand-Rolled Code

Prefer a well-established library over writing your own helper whenever one already covers the need. Priority: standard library first, then a trusted third-party library. Only hand-roll when neither provides it.

- Use stdlib when it covers the need (e.g. `strings.Split` over a hand-rolled splitter, `uuid.NewString()` over a hand-decoded UUID).
- Use `github.com/samber/lo` for common slice/map helpers (e.g. `lo.Filter`, `lo.UniqBy`, `lo.FlatMap`, `lo.FilterMap`) instead of a hand-rolled loop.
- Compile regexps once at package level: `var nameRegex = regexp.MustCompile(...)`, never inside a function. One package-level regexp beats a hand-rolled character loop for format validation.
- Do not reimplement what a dependency already provides.

## Values Over Pointers

Return plain values (not pointers) for data structures and config-like types. They are data, so values are simpler to work with and avoid nil handling.

```go
// bad
func (c *Client) Job(id string) (*Job, error) {
	return nil, fmt.Errorf("get job: %w", err)
}

// good
func (c *Client) Job(id string) (Job, error) {
	return Job{}, fmt.Errorf("get job: %w", err)
}
```

Use a pointer only when you genuinely need to mutate or share the same instance.

## Keep It Minimal

No speculative features, no abstractions that weren't requested, no boilerplate. Write the minimum code that solves the problem. Match existing project patterns.

Within a package, split files by domain rather than size: more, smaller files that each own one domain (e.g. auth, transport, types) over one large file growing without limit.

## End Every File With a Newline

Every file you write or edit ends with exactly one newline. No missing final newline, no extra blank line at the end. `gofmt` writes exactly one: run the formatter and keep its output.

## Common Mistakes

- Shortening variable names to save typing — names must describe meaning.
- Assigning inside an `if`/`switch` condition (`if x := ...; ...`) — assign first, then check.
- Putting the happy path in an `if` branch and the error in `else` — check the error first, happy path flows down the left margin.
- Returning `err` bare or wrapping with `%v` — always `fmt.Errorf("context: %w", err)`.
- Nesting function calls (`outer(inner(...))`) — assign intermediates to named variables first (exceptions: `fmt.Errorf`, a single operation like `len(s)`).
- Hand-rolling a slice/map helper (filter, dedup, flatten) when `lo` already provides it — prefer `lo.Filter`, `lo.UniqBy`, `lo.FlatMap`, `lo.FilterMap`.
- Adding a dependency that duplicates stdlib (e.g. using `lo` when `strings`/`slices`/`maps` already covers it).
- Returning pointers for data structures and configs when a value would do.
- Running logical steps together with no blank-line separation — use a blank line between distinct blocks.
- Squeezing a multi-argument call onto one long line — split it one argument per line when it doesn't read clearly inline.
- Splitting two- or three-argument calls one argument per line — split only 4+ argument or over-long calls.
- `queue.NewQueue()` or `http.HTTPClient()` — the package name is already at the call site.
- `GetStatusName()` — drop the `Get`; keep `Is`/`Has`/`Can` on boolean predicates.
- A real enum value at iota 0 — start with `StatusUnknown` so the zero value means unset.
- Defining an interface beside its single implementation — start concrete, extract when a second consumer appears.
- Defining an interface in the implementor package — the consumer package owns the contract.
- Returning an interface from a constructor — return the concrete struct.
- Sentinel errors without the package prefix, or capitalized error strings — `errors.New("jobstore: not found")`, all lowercase.
- Hand-rolling a character-loop validator or compiling a regexp inside a function — one package-level `regexp.MustCompile`.
- `Url`, `Http` mixed-case acronyms — all caps or all lower.
- Adding explanatory comments instead of making the code self-explanatory.
