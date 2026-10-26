# Python

Apply to all new and edited Python code; flag violations when reviewing. A project's own linter config or `AGENTS.md` wins on conflict.

## Split brackets: add a trailing comma

Always one line, or one item per line. Never the middle form: opening bracket, all items packed on one indented line, closing bracket. If the items do not fit on one line, add a trailing comma after the last item, then run the formatter (`ruff format`). The trailing comma makes the formatter put one item per line.

```python
# Bad
job_input = SubmitJobInput(
    queue=queue, payload=payload, priority=Priority.HIGH
)

def submit_job(
    queue: Queue, payload: dict[str, str], priority: int, timeout: timedelta
) -> Job: ...

# Good
job_input = SubmitJobInput(
    queue=queue,
    payload=payload,
    priority=Priority.HIGH,
)

def submit_job(
    queue: Queue,
    payload: dict[str, str],
    priority: int,
    timeout: timedelta,
) -> Job: ...
```

The same applies to every bracketed list of items: call arguments, `def` parameters, list/dict/set/tuple literals, imports, and parenthesised `with` items.

## Type everything

- Every parameter and every return has an annotation. A function that returns nothing is `-> None`, never unannotated.
- Annotate module-level constants and any local the type checker cannot infer (empty collections, `None` initial values).
- Avoid `Any`. Use a precise type, a `Protocol`, or a pydantic model.
- Use modern syntax: `list[str]`, `str | None`.

```python
def cancel_job(conn: Connection, job_id: uuid.UUID) -> None: ...
```

## Pydantic when it earns its keep

Use `pydantic.BaseModel` when the data needs what pydantic gives: validation, parsing or coercion, serialization, or it crosses a boundary (handler inputs and results, API payloads, config, anything read from outside the program).

- Put validation in `field_validator` / `model_validator`, not ad-hoc `if`/`raise` in callers.
- Use `Field(default_factory=...)` when a default must be built fresh.
- Do not add `ConfigDict` or `frozen=True` unless something needs it.

Do NOT convert a simple internal record. A small `NamedTuple` or `dataclass` that only groups a few trusted values inside one module stays as it is. Do not rewrite existing code to pydantic unless asked. Leave generated code alone.

For a new plain `dict` or positional tuple that carries named fields, use a `NamedTuple` or `dataclass`, or pydantic if it meets the bar above.

## Common Mistakes

- Packing the items on one indented line between the brackets instead of adding the trailing comma.
- Hand-formatting to look like the formatter output instead of running the formatter.
- Leaving `-> None` off a function because "it returns nothing".
- Converting a working `NamedTuple` or `dataclass` to pydantic with no validation, parsing or boundary need.
