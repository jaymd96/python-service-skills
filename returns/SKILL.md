---
name: returns
description: Typed, composable error handling with Result, Maybe, and IO containers. Use when implementing railway-oriented programming, replacing exceptions with typed errors, or using monadic composition. Triggers on Result type, Maybe, railway programming, typed errors, returns library, Success/Failure, monads.
---

# returns — Typed Error Handling (v0.23.0)

## Quick Start

```bash
pip install returns
```

```python
from returns.result import Result, Success, Failure

def parse_int(s: str) -> Result[int, str]:
    try:
        return Success(int(s))
    except ValueError:
        return Failure(f"Cannot parse: {s}")

parse_int("42")   # Success(42)
parse_int("abc")  # Failure("Cannot parse: abc")
```

## Key Patterns

### Chaining with bind/map
```python
result = (
    parse_int("42")
    .map(lambda x: x * 2)        # transform Success value
    .bind(validate_positive)       # chain operations that return Result
    .value_or(0)                   # extract with default
)
```

### @safe decorator (exceptions -> Result)
```python
from returns.result import safe

@safe
def divide(a: int, b: int) -> float:
    return a / b

divide(10, 0)  # Failure(ZeroDivisionError(...))
divide(10, 2)  # Success(5.0)
```

### flow (pipe-like composition)
```python
from returns.pipeline import flow
from returns.pointfree import bind, map_

result = flow(
    raw_input,
    parse_request,
    bind(validate),
    bind(process),
    map_(serialize),
)
```

## References

- **[api.md](references/api.md)** — All containers (Result, Maybe, IO, IOResult, Future, RequiresContext), operations, pattern matching, pointfree functions, and do notation
- **[examples.md](references/examples.md)** — Gotchas (mypy plugin, @safe behavior, performance), and complete code examples
