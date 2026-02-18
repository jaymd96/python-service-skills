# returns

> Typed, composable, railway-oriented error handling for Python.

`returns` (by [dry-python](https://github.com/dry-python)) brings **monadic containers** to Python with full type safety. Instead of raising exceptions or returning `None`, you wrap values in containers like `Result`, `Maybe`, and `IO` and chain operations on them -- failures propagate automatically without breaking the pipeline.

| Detail | Value |
|---|---|
| **PyPI** | [returns](https://pypi.org/project/returns/) |
| **Latest stable version** | **0.23.0** (verify at PyPI for newer releases) |
| **Python** | 3.9 + |
| **Source** | [github.com/dry-python/returns](https://github.com/dry-python/returns) |
| **Docs** | [returns.readthedocs.io](https://returns.readthedocs.io) |
| **License** | BSD-2-Clause |

---

## Table of Contents

1. [Installation](#installation)
2. [Core Containers](#core-containers)
3. [Key Operations](#key-operations)
4. [Pattern Matching](#pattern-matching)
5. [Pointfree Functions](#pointfree-functions)
6. [Do Notation](#do-notation)
7. [Type Safety](#type-safety)
8. [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
9. [Complete Code Examples](#complete-code-examples)

---

## Installation

```bash
pip install returns
```

To enable the **mypy plugin** (strongly recommended), add this to your `mypy.ini` or `setup.cfg`:

```ini
[mypy]
plugins =
    returns.contrib.mypy.returns_plugin
```

Or in `pyproject.toml`:

```toml
[tool.mypy]
plugins = ["returns.contrib.mypy.returns_plugin"]
```

---

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

---

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

---

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

---

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

---

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

---

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

---

## Gotchas and Common Mistakes

### 1. Forgetting to Handle the Failure Case

The most common mistake is treating a `Result` as if it is always a `Success`. If you call `.unwrap()` without checking, you will get a runtime `UnwrapFailedError`.

```python
# BAD: crashes if result is a Failure
value = some_result.unwrap()

# GOOD: provide a default
value = some_result.value_or(default_value)

# GOOD: pattern match or isinstance check
match some_result:
    case Success(value):
        process(value)
    case Failure(error):
        handle_error(error)
```

### 2. Type Checker Configuration

Without the mypy plugin, you will see confusing type errors. The plugin is not optional in practice -- always configure it.

```ini
# REQUIRED in mypy config:
[mypy]
plugins = returns.contrib.mypy.returns_plugin
```

If you use **pyright** or **pylance** instead of mypy, be aware that full support requires the mypy plugin. Some type features may not work perfectly with other checkers.

### 3. Performance Overhead

Every `.map()` and `.bind()` call creates a new container object. For hot loops processing millions of items, this overhead matters.

```python
# Potentially slow for very large datasets:
result = Success(data)
for transform in huge_list_of_transforms:
    result = result.bind(transform)

# Consider plain Python for tight loops:
try:
    value = data
    for transform in huge_list_of_transforms:
        value = transform(value)
except SomeError as e:
    handle(e)
```

**Rule of thumb:** Use `returns` at the service/application layer for control flow and error propagation. Use plain Python in compute-intensive inner loops.

### 4. When NOT to Use returns

`returns` adds value for **complex error propagation** and **composable pipelines**. It is overkill for simple cases:

```python
# OVERKILL -- just use a simple if/else:
def is_even(n: int) -> Result[bool, str]:
    if n % 2 == 0:
        return Success(True)
    return Failure("odd number")

# JUST DO THIS:
def is_even(n: int) -> bool:
    return n % 2 == 0
```

Use `returns` when:
- You have **chains** of operations that can each fail.
- You want to **compose** functions in pipelines.
- You care about **typed error handling** (knowing exactly what errors a function can produce).
- You want to **separate pure and impure** code at the type level.

Avoid `returns` when:
- A simple `try/except` or `if/else` suffices.
- You are writing performance-critical numeric code.
- Your team is unfamiliar with monadic patterns and the learning curve is not justified.

### 5. Interaction with Exceptions

`returns` does **not** eliminate exceptions from Python. Third-party libraries, the standard library, and your own code can still raise. Use `@safe` / `@impure_safe` at the boundary to convert exceptions into `Result`/`IOResult`:

```python
from returns.result import safe
from returns.io import impure_safe
import json

# Convert a function that raises into one that returns Result:
@safe
def parse_json(raw: str) -> dict:
    return json.loads(raw)  # may raise JSONDecodeError

# For I/O operations:
@impure_safe
def read_file(path: str) -> str:
    with open(path) as f:
        return f.read()  # may raise IOError
```

**Important:** `@safe` catches **all** exceptions (`Exception` subclasses). If you need to catch only specific exceptions, handle them manually:

```python
from returns.result import Result, Success, Failure

def parse_int_strict(s: str) -> Result[int, ValueError]:
    try:
        return Success(int(s))
    except ValueError as e:
        return Failure(e)
    # Other exceptions (TypeError, etc.) will still propagate
```

---

## Complete Code Examples

### Example 1: Basic Result Usage for Error Handling

A user registration flow where each step can fail with a descriptive error:

```python
from returns.result import Result, Success, Failure


def validate_username(name: str) -> Result[str, str]:
    if len(name) < 3:
        return Failure("Username must be at least 3 characters")
    if not name.isalnum():
        return Failure("Username must be alphanumeric")
    return Success(name.lower())


def validate_email(email: str) -> Result[str, str]:
    if "@" not in email or "." not in email:
        return Failure("Invalid email format")
    return Success(email.lower())


def validate_password(password: str) -> Result[str, str]:
    if len(password) < 8:
        return Failure("Password must be at least 8 characters")
    if password.isalpha() or password.isdigit():
        return Failure("Password must contain both letters and digits")
    return Success(password)


# Using the validators:
username = validate_username("Al")
# Failure('Username must be at least 3 characters')

username = validate_username("Alice123")
# Success('alice123')

email = validate_email("alice@example.com")
# Success('alice@example.com')

password = validate_password("short")
# Failure('Password must be at least 8 characters')
```

### Example 2: Chaining Operations with bind

Building a data processing pipeline where each step depends on the previous one:

```python
from returns.result import Result, Success, Failure
from dataclasses import dataclass


@dataclass
class User:
    name: str
    age: int
    email: str


def parse_name(raw: dict) -> Result[str, str]:
    name = raw.get("name")
    if not name or not isinstance(name, str):
        return Failure("missing or invalid 'name'")
    return Success(name.strip())


def parse_age(raw: dict) -> Result[int, str]:
    age = raw.get("age")
    if age is None:
        return Failure("missing 'age'")
    if not isinstance(age, int) or age < 0 or age > 150:
        return Failure(f"invalid age: {age}")
    return Success(age)


def parse_email(raw: dict) -> Result[str, str]:
    email = raw.get("email")
    if not email or "@" not in str(email):
        return Failure("missing or invalid 'email'")
    return Success(str(email).lower())


def parse_user(raw: dict) -> Result[User, str]:
    """Parse and validate a user from a raw dictionary using do notation."""
    return Result.do(
        User(name=name, age=age, email=email)
        for name in parse_name(raw)
        for age in parse_age(raw)
        for email in parse_email(raw)
    )


# Valid input:
user = parse_user({"name": "Alice", "age": 30, "email": "Alice@Example.COM"})
# Success(User(name='Alice', age=30, email='alice@example.com'))

# Invalid input -- short-circuits at first failure:
user = parse_user({"name": "", "age": 30, "email": "a@b.com"})
# Failure("missing or invalid 'name'")

user = parse_user({"name": "Alice", "age": -5, "email": "a@b.com"})
# Failure('invalid age: -5')
```

### Example 3: HTTP Request Handling with IOResult

Fetching and processing data from an API, where every step is impure and can fail:

```python
from returns.io import IOResult, IOSuccess, IOFailure, impure_safe
from returns.result import safe
from returns.pipeline import flow
from returns.pointfree import bind, map_
import json
from urllib.request import urlopen
from urllib.error import URLError
from dataclasses import dataclass


@dataclass
class Repository:
    name: str
    stars: int
    language: str


@impure_safe
def fetch_url(url: str) -> str:
    """Fetch raw content from a URL. May raise URLError, etc."""
    with urlopen(url) as response:
        return response.read().decode("utf-8")


@safe
def parse_json_body(raw: str) -> dict:
    """Parse JSON string. May raise JSONDecodeError."""
    return json.loads(raw)


def extract_repo(data: dict) -> IOResult[Repository, Exception]:
    """Extract repository info from parsed JSON."""
    try:
        repo = Repository(
            name=data["full_name"],
            stars=data["stargazers_count"],
            language=data.get("language", "Unknown"),
        )
        return IOSuccess(repo)
    except (KeyError, TypeError) as e:
        return IOFailure(e)


def fetch_repo_info(owner: str, repo: str) -> IOResult[Repository, Exception]:
    """Full pipeline: fetch -> parse -> extract."""
    url = f"https://api.github.com/repos/{owner}/{repo}"
    return fetch_url(url).bind(
        lambda raw: IOResult.from_result(parse_json_body(raw))
    ).bind(extract_repo)


# Usage:
result = fetch_repo_info("dry-python", "returns")
# IOSuccess(Repository(name='dry-python/returns', stars=..., language='Python'))
# or IOFailure(URLError(...)) if the network call fails
```

### Example 4: Data Validation Pipeline

A composable validation pipeline using `flow` and pointfree functions:

```python
from returns.result import Result, Success, Failure
from returns.pipeline import flow
from returns.pointfree import bind, map_
from typing import TypeAlias

# Define a type alias for our validation result
ValidationResult: TypeAlias = Result[dict, str]


def require_field(field: str):
    """Return a validator that checks a field exists."""
    def _validate(data: dict) -> Result[dict, str]:
        if field not in data or data[field] is None:
            return Failure(f"missing required field: '{field}'")
        return Success(data)
    return _validate


def validate_type(field: str, expected_type: type):
    """Return a validator that checks a field's type."""
    def _validate(data: dict) -> Result[dict, str]:
        value = data.get(field)
        if value is not None and not isinstance(value, expected_type):
            return Failure(
                f"'{field}' must be {expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        return Success(data)
    return _validate


def validate_range(field: str, min_val: float, max_val: float):
    """Return a validator that checks a numeric field is in range."""
    def _validate(data: dict) -> Result[dict, str]:
        value = data.get(field)
        if value is not None and not (min_val <= value <= max_val):
            return Failure(f"'{field}' must be between {min_val} and {max_val}")
        return Success(data)
    return _validate


def validate_order(data: dict) -> ValidationResult:
    """Validate an order dictionary through a pipeline."""
    return flow(
        Success(data),
        bind(require_field("product")),
        bind(require_field("quantity")),
        bind(validate_type("quantity", int)),
        bind(validate_range("quantity", 1, 1000)),
        bind(require_field("price")),
        bind(validate_type("price", (int, float))),
        bind(validate_range("price", 0.01, 99999.99)),
    )


# Valid order:
order = validate_order({
    "product": "Widget",
    "quantity": 5,
    "price": 19.99,
})
# Success({'product': 'Widget', 'quantity': 5, 'price': 19.99})

# Missing field:
order = validate_order({"product": "Widget"})
# Failure("missing required field: 'quantity'")

# Out of range:
order = validate_order({
    "product": "Widget",
    "quantity": 5000,
    "price": 19.99,
})
# Failure("'quantity' must be between 1 and 1000")
```

### Example 5: Combining with attrs/cattrs

Using `returns` alongside `attrs` for structured data with validation:

```python
import attr
from returns.result import Result, Success, Failure, safe
from returns.pipeline import flow
from returns.pointfree import bind, map_


@attr.s(auto_attribs=True, frozen=True)
class Address:
    street: str
    city: str
    zip_code: str


@attr.s(auto_attribs=True, frozen=True)
class Customer:
    name: str
    email: str
    address: Address


def validate_zip(zip_code: str) -> Result[str, str]:
    """Validate a US zip code format."""
    cleaned = zip_code.strip()
    if len(cleaned) == 5 and cleaned.isdigit():
        return Success(cleaned)
    if len(cleaned) == 10 and cleaned[5] == "-":
        return Success(cleaned)
    return Failure(f"invalid zip code: {zip_code!r}")


def validate_email(email: str) -> Result[str, str]:
    """Basic email validation."""
    if "@" in email and "." in email.split("@")[-1]:
        return Success(email.lower().strip())
    return Failure(f"invalid email: {email!r}")


def build_address(data: dict) -> Result[Address, str]:
    """Build a validated Address from raw data."""
    return Result.do(
        Address(street=street, city=city, zip_code=zip_code)
        for street in (
            Success(data["street"])
            if data.get("street")
            else Failure("missing street")
        )
        for city in (
            Success(data["city"])
            if data.get("city")
            else Failure("missing city")
        )
        for zip_code in validate_zip(data.get("zip_code", ""))
    )


def build_customer(data: dict) -> Result[Customer, str]:
    """Build a validated Customer from raw data."""
    return Result.do(
        Customer(name=name, email=email, address=address)
        for name in (
            Success(data["name"])
            if data.get("name")
            else Failure("missing name")
        )
        for email in validate_email(data.get("email", ""))
        for address in build_address(data.get("address", {}))
    )


# Valid customer:
customer = build_customer({
    "name": "Alice Smith",
    "email": "Alice@Example.COM",
    "address": {
        "street": "123 Main St",
        "city": "Springfield",
        "zip_code": "62704",
    },
})
# Success(Customer(
#     name='Alice Smith',
#     email='alice@example.com',
#     address=Address(street='123 Main St', city='Springfield', zip_code='62704')
# ))

# Invalid zip code:
customer = build_customer({
    "name": "Bob",
    "email": "bob@test.com",
    "address": {
        "street": "456 Oak Ave",
        "city": "Portland",
        "zip_code": "ABCDE",
    },
})
# Failure("invalid zip code: 'ABCDE'")
```

---

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

---

## Further Reading

- [Official documentation](https://returns.readthedocs.io/en/latest/)
- [GitHub repository](https://github.com/dry-python/returns)
- [PyPI page](https://pypi.org/project/returns/)
- [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) -- the F# concept that inspired this library
