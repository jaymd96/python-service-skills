# returns — API Reference

> Part of the returns skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core Containers](#core-containers)
  - [Result\[ValueType, ErrorType\]](#resultvaluetype-errortype----successfailure-for-error-handling)
  - [Maybe\[ValueType\]](#maybevaluetype----optional-values-without-none)
  - [IO\[ValueType\]](#iovaluetype----marking-impure-operations)
  - [IOResult\[ValueType, ErrorType\]](#ioresultvaluetype-errortype----combining-io--result)
  - [Future and FutureResult](#futurevaluetype-and-futureresultvaluetype-errortype----async-containers)
  - [RequiresContext](#requirescontextvaluetype-envtype----dependency-injection-via-reader-monad)
- [Key Operations](#key-operations)
  - [.map(func)](#mapfunc----transform-the-inner-value)
  - [.bind(func)](#bindfunc----chain-operations-returning-containers)
  - [.alt(func)](#altfunc----handle-the-failure-case)
  - [.lash(func)](#lashfunc----recover-from-failures)
  - [.value_or(default)](#value_ordefault----extract-with-a-default)
  - [.unwrap()](#unwrap----unsafe-extraction)
  - [flow()](#flow----pipe-like-composition)
  - [@safe and @impure_safe](#safe-and-impure_safe----exception-catching-decorators)
- [Pattern Matching](#pattern-matching)
- [Pointfree Functions](#pointfree-functions)
- [Do Notation](#do-notation)
- [Type Safety](#type-safety)
- [Quick Reference](#quick-reference)

## Core Containers

`returns` provides a family of container types, each encoding a specific computational effect. They all share a common interface but model different concerns.

### `Result[ValueType, ErrorType]` -- Success/Failure for Error Handling

`Result` is the most commonly used container. It replaces exceptions with typed, explicit error handling. A `Result` is either a `Success` wrapping a good value or a `Failure` wrapping an error.

```python
from returns.result import Result, Success, Failure

def divide(a: float, b: float) -> Result[float, str]:
    if b == 0:
        return Failure("division by zero")
    return Success(a / b)

result = divide(10, 3)
# Success(3.3333...)

result = divide(10, 0)
# Failure('division by zero')
```

**Key properties:**

- `Success(value)` -- the happy path; wraps a successful value.
- `Failure(error)` -- the sad path; wraps an error value.
- Operations like `.map()` and `.bind()` act on the `Success` branch; `Failure` passes through untouched (railway-oriented programming).

### `Maybe[ValueType]` -- Optional Values Without None

`Maybe` models the presence or absence of a value, replacing bare `None` checks.

```python
from returns.maybe import Maybe, Some, Nothing

def find_user(user_id: int) -> Maybe[str]:
    users = {1: "Alice", 2: "Bob"}
    if user_id in users:
        return Some(users[user_id])
    return Nothing

name = find_user(1)   # Some('Alice')
name = find_user(99)  # Nothing
```

**Key properties:**

- `Some(value)` -- a value is present.
- `Nothing` -- a singleton representing absence (no wrapped value).
- `.map()` and `.bind()` act on `Some`; `Nothing` short-circuits.

You can convert between `Maybe` and `Result`:

```python
from returns.maybe import Maybe, Some, Nothing
from returns.result import Result

maybe_val: Maybe[int] = Some(42)
result_val: Result[int, None] = maybe_val.to_result()
# Success(42)

nothing: Maybe[int] = Nothing
result_nothing: Result[int, None] = nothing.to_result()
# Failure(None)
```

### `IO[ValueType]` -- Marking Impure Operations

`IO` wraps values that came from side effects (file reads, random numbers, current time). It does not prevent the side effect -- it marks it in the type system so pure and impure code stay separate.

```python
from returns.io import IO

def read_config() -> IO[str]:
    # Side effect: reading from filesystem
    with open("config.txt") as f:
        return IO(f.read())

config: IO[str] = read_config()

# Transform the impure value without unwrapping
upper_config: IO[str] = config.map(str.upper)
```

**Key properties:**

- `IO(value)` -- wraps an impure value.
- You cannot accidentally unwrap it without calling `.unwrap()` explicitly, which signals impurity.
- Encourages keeping your core logic pure.

### `IOResult[ValueType, ErrorType]` -- Combining IO + Result

`IOResult` combines the effects of `IO` (impurity) and `Result` (fallibility). This is the workhorse container for real-world code that does I/O and can fail.

```python
from returns.io import IOResult, IOSuccess, IOFailure

def fetch_config(path: str) -> IOResult[str, OSError]:
    try:
        with open(path) as f:
            return IOSuccess(f.read())
    except OSError as e:
        return IOFailure(e)

config = fetch_config("/etc/app.conf")
# IOSuccess('...file contents...') or IOFailure(OSError(...))
```

**Key properties:**

- `IOSuccess(value)` -- impure computation that succeeded.
- `IOFailure(error)` -- impure computation that failed.
- Combines the semantics of `IO` and `Result` in a single container.

### `Future[ValueType]` and `FutureResult[ValueType, ErrorType]` -- Async Containers

`Future` and `FutureResult` are the async counterparts of `IO` and `IOResult`. They wrap coroutines in a composable, typed container.

```python
import httpx
from returns.future import FutureResultE, future_safe

@future_safe
async def fetch_url(url: str) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.text

# Returns FutureResultE[str] (alias for FutureResult[str, Exception])
page: FutureResultE[str] = fetch_url("https://example.com")
```

**Key properties:**

- `Future` wraps an async value (like `IO` but async).
- `FutureResult` wraps an async value that can fail (like `IOResult` but async).
- You `await` the container to get the inner `IOResult` or `IO`.
- `@future_safe` decorator catches exceptions and wraps them in `FutureResult`.

### `RequiresContext[ValueType, EnvType]` -- Dependency Injection via Reader Monad

`RequiresContext` (and its variants `RequiresContextResult`, `RequiresContextIOResult`, etc.) lets you defer the injection of dependencies. Your functions declare what environment they need, and you supply it later.

```python
from returns.context import RequiresContext

def get_greeting(name: str) -> RequiresContext[str, dict]:
    """Needs a config dict to produce a greeting."""
    return RequiresContext(
        lambda config: f"{config['prefix']} {name}!"
    )

greeting = get_greeting("Alice")

# Supply the dependency later:
result = greeting({"prefix": "Hello"})
# 'Hello Alice!'
```

**Related context containers:**

| Container | Effect |
|---|---|
| `RequiresContext` | Pure reader |
| `RequiresContextResult` | Reader + fallibility |
| `RequiresContextIOResult` | Reader + fallibility + I/O |
| `RequiresContextFutureResult` | Reader + fallibility + async I/O |

## Key Operations

All containers in `returns` share a consistent interface. The most important operations are listed below.

### `.map(func)` -- Transform the Inner Value

Applies a **plain function** to the wrapped value. The function takes a raw value and returns a raw value.

```python
from returns.result import Success, Failure

Success(5).map(lambda x: x * 2)    # Success(10)
Failure("err").map(lambda x: x * 2) # Failure('err') -- unchanged
```

### `.bind(func)` -- Chain Operations Returning Containers

Applies a function that **itself returns a container**. This is how you chain fallible operations.

```python
from returns.result import Result, Success, Failure

def parse_int(s: str) -> Result[int, str]:
    try:
        return Success(int(s))
    except ValueError:
        return Failure(f"not an integer: {s!r}")

def check_positive(n: int) -> Result[int, str]:
    if n > 0:
        return Success(n)
    return Failure(f"{n} is not positive")

# Chain them:
result = Success("42").bind(parse_int).bind(check_positive)
# Success(42)

result = Success("-5").bind(parse_int).bind(check_positive)
# Failure('-5 is not positive')

result = Success("abc").bind(parse_int).bind(check_positive)
# Failure("not an integer: 'abc'")
```

### `.alt(func)` -- Handle the Failure Case

Applies a function to the **failure** value (the opposite of `.map()`).

```python
from returns.result import Success, Failure

Failure("err").alt(lambda e: f"Error: {e}")
# Failure('Error: err')

Success(42).alt(lambda e: f"Error: {e}")
# Success(42) -- unchanged
```

### `.lash(func)` -- Recover from Failures

Like `.bind()` but for the failure case. The function receives the error and returns a new container, allowing recovery.

```python
from returns.result import Result, Success, Failure

def recover(error: str) -> Result[int, str]:
    if error == "not found":
        return Success(0)  # default value
    return Failure(error)   # re-fail

Failure("not found").lash(recover)  # Success(0)
Failure("fatal").lash(recover)      # Failure('fatal')
Success(42).lash(recover)           # Success(42) -- unchanged
```

### `.value_or(default)` -- Extract with a Default

Extracts the success value, or returns the default if it is a failure.

```python
from returns.result import Success, Failure

Success(42).value_or(0)   # 42
Failure("err").value_or(0) # 0
```

### `.unwrap()` -- Unsafe Extraction

Extracts the success value, but **raises `UnwrapFailedError`** if the container holds a failure. Use sparingly and only at the boundary of your program.

```python
from returns.result import Success, Failure

Success(42).unwrap()    # 42
Failure("err").unwrap() # raises UnwrapFailedError
```

There is also `.failure()` which does the opposite -- unwraps the failure value, raising if it is a success.

### `flow()` -- Pipe-like Composition

`flow` passes a value through a series of functions, left to right. It is like a Unix pipe for Python functions.

```python
from returns.pipeline import flow
from returns.pointfree import bind
from returns.result import Success

def double(x: int) -> int:
    return x * 2

def to_str(x: int) -> str:
    return str(x)

result = flow(
    10,
    double,
    double,
    to_str,
)
# '40'
```

`flow` works with both plain functions and container-aware functions (when combined with pointfree operations):

```python
from returns.pipeline import flow
from returns.pointfree import bind
from returns.result import Result, Success, Failure

def validate(x: int) -> Result[int, str]:
    if x > 0:
        return Success(x)
    return Failure("must be positive")

def double_result(x: int) -> Result[int, str]:
    return Success(x * 2)

result = flow(
    Success(5),
    bind(validate),
    bind(double_result),
)
# Success(10)
```

### `@safe` and `@impure_safe` -- Exception-Catching Decorators

These decorators wrap functions that may raise exceptions, catching them and returning `Result` or `IOResult` instead.

```python
from returns.result import safe

@safe
def parse_json(raw: str) -> dict:
    import json
    return json.loads(raw)

parse_json('{"a": 1}')   # Success({'a': 1})
parse_json('not json')    # Failure(JSONDecodeError(...))
```

```python
from returns.io import impure_safe

@impure_safe
def read_file(path: str) -> str:
    with open(path) as f:
        return f.read()

read_file("exists.txt")     # IOSuccess('...')
read_file("missing.txt")    # IOFailure(FileNotFoundError(...))
```

## Pattern Matching

Python 3.10+ structural pattern matching works with `returns` containers:

```python
from returns.result import Result, Success, Failure

def describe(result: Result[int, str]) -> str:
    match result:
        case Success(value):
            return f"Got value: {value}"
        case Failure(error):
            return f"Got error: {error}"

describe(Success(42))       # 'Got value: 42'
describe(Failure("oops"))   # 'Got error: oops'
```

This also works with `Maybe`:

```python
from returns.maybe import Maybe, Some, Nothing

def greet(name: Maybe[str]) -> str:
    match name:
        case Some(n):
            return f"Hello, {n}!"
        case Nothing:
            return "Hello, stranger!"
```

## Pointfree Functions

Pointfree functions are standalone versions of container methods. They are designed to be passed into `flow()`, `.map()`, and other higher-order contexts where you need a callable rather than a method call.

```python
from returns.pointfree import bind, map_, alt, lash
from returns.result import Result, Success, Failure

def add_one(x: int) -> Result[int, str]:
    return Success(x + 1)

# These are equivalent:
Success(5).bind(add_one)      # Success(6)
bind(add_one)(Success(5))     # Success(6)
```

Common pointfree functions:

| Function | Equivalent method | Description |
|---|---|---|
| `bind(f)` | `.bind(f)` | Chain a function returning a container |
| `map_(f)` | `.map(f)` | Transform the inner value |
| `alt(f)` | `.alt(f)` | Transform the failure value |
| `lash(f)` | `.lash(f)` | Recover from failure with a new container |

These are importable from `returns.pointfree`.

Pointfree functions shine in pipelines:

```python
from returns.pipeline import flow
from returns.pointfree import bind, map_
from returns.result import Result, Success

def validate_age(age: int) -> Result[int, str]:
    if 0 < age < 150:
        return Success(age)
    return Failure(f"invalid age: {age}")

def format_age(age: int) -> str:
    return f"{age} years old"

result = flow(
    Success(25),
    bind(validate_age),
    map_(format_age),
)
# Success('25 years old')
```

## Do Notation

Do notation lets you write monadic code in an **imperative style** using generators. Instead of deeply nested `.bind()` chains, you use `yield` to "unwrap" intermediate results.

```python
from returns.result import Result, Success, Failure
from returns.pointfree import bind

def parse_int(s: str) -> Result[int, str]:
    try:
        return Success(int(s))
    except ValueError:
        return Failure(f"not an integer: {s!r}")

def check_range(n: int, lo: int, hi: int) -> Result[int, str]:
    if lo <= n <= hi:
        return Success(n)
    return Failure(f"{n} not in [{lo}, {hi}]")

# Using Result.do:
result: Result[str, str] = Result.do(
    f"Valid port: {port}"
    for port in parse_int("8080")
    for port in check_range(port, 1, 65535)
)
# Success('Valid port: 8080')
```

If any step yields a `Failure`, the entire `do` block short-circuits:

```python
result: Result[str, str] = Result.do(
    f"Valid port: {port}"
    for port in parse_int("abc")
    for port in check_range(port, 1, 65535)
)
# Failure("not an integer: 'abc'")
```

This pattern is available on `Result`, `Maybe`, `IOResult`, `FutureResult`, and context containers.

## Type Safety

### The mypy Plugin

`returns` ships with a mypy plugin that provides:

- **Correct type narrowing** for `Success`/`Failure`, `Some`/`Nothing`.
- **Higher-kinded type support** for generic container operations.
- **Proper inference** for `flow()`, `bind()`, and other compositional helpers.

**Setup** (add to your mypy config):

```ini
[mypy]
plugins =
    returns.contrib.mypy.returns_plugin
```

### Type Narrowing

After a type check, mypy understands which branch you are in:

```python
from returns.result import Result, Success, Failure

def process(result: Result[int, str]) -> int:
    if isinstance(result, Success):
        # mypy knows result is Success[int, str] here
        return result.unwrap() * 2
    else:
        # mypy knows result is Failure[int, str] here
        print(f"Error: {result.failure()}")
        return -1
```

### TypeVar Generics

Container types are generic:

```python
from returns.result import Result, Success, Failure

# Explicit annotation
x: Result[int, str] = Success(42)
y: Result[int, str] = Failure("boom")

# Type inference works too:
z = Success(42)  # inferred as Success[int]
```

## Quick Reference

| Container | Success type | Failure type | Use case |
|---|---|---|---|
| `Result` | `Success` | `Failure` | Synchronous, pure error handling |
| `Maybe` | `Some` | `Nothing` | Optional values |
| `IO` | `IO` | -- | Marking impure values |
| `IOResult` | `IOSuccess` | `IOFailure` | Impure + fallible operations |
| `Future` | `Future` | -- | Async impure values |
| `FutureResult` | `FutureSuccess` | `FutureFailure` | Async impure + fallible |
| `RequiresContext` | -- | -- | Dependency injection |

| Operation | Acts on | Purpose |
|---|---|---|
| `.map(f)` | Success value | Transform inner value |
| `.bind(f)` | Success value | Chain container-returning functions |
| `.alt(f)` | Failure value | Transform the error |
| `.lash(f)` | Failure value | Recover from failure |
| `.value_or(d)` | Either | Extract with default |
| `.unwrap()` | Success only | Unsafe extraction (raises on failure) |
| `@safe` | Exceptions | Convert exceptions to `Result` |
| `@impure_safe` | Exceptions | Convert exceptions to `IOResult` |
| `flow(v, *fns)` | Value | Pipe value through functions |
