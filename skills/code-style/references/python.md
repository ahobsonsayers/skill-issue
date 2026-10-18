# Python

Apply to all new and edited Python code; flag violations when reviewing. A project's own linter config or `AGENTS.md` wins on conflict.

## Split brackets: add a trailing comma

If a call, definition, or collection does not fit on one line, never leave the arguments packed together between the brackets. Add a trailing comma after the last argument, then run the formatter (`ruff format`). The trailing comma makes the formatter put one argument per line.

```python
# Bad
outcome = create_outcome(
    conn, CreateOutcomeInput(project_id=project_id, due_date=due_date, outcome_type=OutcomeType.PDC)
)

# Good
outcome = create_outcome(
    conn,
    CreateOutcomeInput(
        project_id=project_id,
        due_date=due_date,
        outcome_type=OutcomeType.PDC,
    ),
)
```

The same applies to `def` signatures, list/dict/set literals, and imports.

## Type everything

- Every parameter and every return has an annotation. A function that returns nothing is `-> None`, never unannotated.
- Annotate module-level constants and any local the type checker cannot infer (empty collections, `None` initial values).
- Avoid `Any`. Use a precise type, a `Protocol`, or a pydantic model.
- Use modern syntax: `list[str]`, `str | None`.

```python
def archive_project(conn: Connection, project_id: uuid.UUID) -> None: ...
```

## Pydantic when it earns its keep

Use `pydantic.BaseModel` when the data needs what pydantic gives: validation, parsing or coercion, serialization, or it crosses a boundary (handler inputs and results, API payloads, config, anything read from outside the program).

- Put validation in `field_validator` / `model_validator`, not ad-hoc `if`/`raise` in callers.
- Use `Field(default_factory=...)` when a default must be built fresh.
- Do not add `ConfigDict` or `frozen=True` unless something needs it.

Do NOT convert a simple internal record. A small `NamedTuple` or `dataclass` that only groups a few trusted values inside one module stays as it is. Do not rewrite existing code to pydantic unless asked. Leave generated code alone.

For a new plain `dict` or positional tuple that carries named fields, use a `NamedTuple` or `dataclass`, or pydantic if it meets the bar above.

## Common Mistakes

- Packing the arguments on one indented line instead of adding the trailing comma.
- Hand-formatting to look like the formatter output instead of running the formatter.
- Leaving `-> None` off a function because "it returns nothing".
- Converting a working `NamedTuple` or `dataclass` to pydantic with no validation, parsing or boundary need.
