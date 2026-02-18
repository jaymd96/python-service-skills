# returns — Examples & Gotchas

> Part of the returns skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Forgetting to Handle the Failure Case](#1-forgetting-to-handle-the-failure-case)
  - [2. Type Checker Configuration](#2-type-checker-configuration)
  - [3. Performance Overhead](#3-performance-overhead)
  - [4. When NOT to Use returns](#4-when-not-to-use-returns)
  - [5. Interaction with Exceptions](#5-interaction-with-exceptions)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Basic Result Usage for Error Handling](#example-1-basic-result-usage-for-error-handling)
  - [Example 2: Chaining Operations with bind](#example-2-chaining-operations-with-bind)
  - [Example 3: HTTP Request Handling with IOResult](#example-3-http-request-handling-with-ioresult)
  - [Example 4: Data Validation Pipeline](#example-4-data-validation-pipeline)
  - [Example 5: Combining with attrs/cattrs](#example-5-combining-with-attrscattrs)
- [Further Reading](#further-reading)

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

## Further Reading

- [Official documentation](https://returns.readthedocs.io/en/latest/)
- [GitHub repository](https://github.com/dry-python/returns)
- [PyPI page](https://pypi.org/project/returns/)
- [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) -- the F# concept that inspired this library
