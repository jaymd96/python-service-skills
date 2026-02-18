# toolz — Examples & Gotchas

> Part of the toolz skill. See [SKILL.md](../SKILL.md) for overview.

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Lazy Evaluation Behavior](#1-lazy-evaluation-behavior)
  - [2. curry with Keyword Arguments](#2-curry-with-keyword-arguments)
  - [3. memoize with Mutable Arguments](#3-memoize-with-mutable-arguments)
  - [4. Performance Considerations vs List Comprehensions](#4-performance-considerations-vs-list-comprehensions)
  - [5. toolz.curried vs Regular toolz](#5-toolzcurried-vs-regular-toolz)
- [Complete Code Examples](#complete-code-examples)
  - [Example 1: Data Pipeline with pipe](#example-1-data-pipeline-with-pipe)
  - [Example 2: Curried Function Composition](#example-2-curried-function-composition)
  - [Example 3: Dictionary Transformations](#example-3-dictionary-transformations)
  - [Example 4: Grouping and Aggregation](#example-4-grouping-and-aggregation)
  - [Example 5: Functional Error Handling with excepts](#example-5-functional-error-handling-with-excepts)
- [Summary](#summary)

## Gotchas and Common Mistakes

### 1. Lazy Evaluation Behavior

Most itertoolz functions return **lazy iterators**, not lists. This means:

```python
from toolz import take, drop, partition

result = take(5, range(1000000))
# result is a list (take is an exception -- it materializes)

result = drop(5, range(10))
# result is a lazy iterator! Must consume it.
list(result)  # [5, 6, 7, 8, 9]

partitions = partition(3, range(9))
# partitions is a lazy iterator of tuples
list(partitions)  # [(0,1,2), (3,4,5), (6,7,8)]
```

**The trap:** iterators can only be consumed once.

```python
from toolz import drop

nums = drop(2, [1, 2, 3, 4, 5])

list(nums)  # [3, 4, 5]
list(nums)  # [] -- already consumed!
```

**Fix:** materialize with `list()` if you need to iterate multiple times, or use `peek()` to inspect without consuming.

### 2. `curry` with Keyword Arguments

`curry` works with keyword arguments, but there are edge cases:

```python
from toolz import curry

@curry
def greet(greeting, name, punctuation="!"):
    return f"{greeting} {name}{punctuation}"

# This works -- positional partial application
hello = greet("Hello")
hello("World")  # "Hello World!"

# This also works -- keyword partial application
greet_politely = greet(punctuation=".")
greet_politely("Hello", "World")  # "Hello World."
```

**The trap:** `curry` inspects the function signature using `inspect`. Functions with `*args` or `**kwargs` cannot be auto-curried because toolz cannot determine when enough arguments have been supplied.

```python
@curry
def bad(*args):  # How many args until we call it? toolz doesn't know
    return sum(args)

# This calls immediately with zero args because toolz can't tell it needs more
bad()  # 0  (called immediately, not curried)
```

**Fix:** give curried functions explicit parameter lists.

### 3. `memoize` with Mutable Arguments

`memoize` hashes arguments to build cache keys. Mutable arguments cause problems:

```python
from toolz import memoize

@memoize
def process(data):
    return sum(data)

process([1, 2, 3])  # TypeError: unhashable type: 'list'
```

**Fix:** use a custom `key` function to convert arguments to something hashable:

```python
@memoize(key=lambda args, kwargs: tuple(args[0]))
def process(data):
    return sum(data)

process([1, 2, 3])  # 6
process([1, 2, 3])  # 6 (cached)
```

Or convert to tuples before calling:

```python
@memoize
def process(data):
    return sum(data)

process(tuple([1, 2, 3]))  # 6
```

### 4. Performance Considerations vs List Comprehensions

toolz has function call overhead. For simple transformations, list comprehensions are faster:

```python
# Faster (no function call overhead):
[x ** 2 for x in range(1000) if x % 2 == 0]

# Slower (function dispatch + iterator protocol):
from toolz import pipe
from toolz.curried import map, filter

pipe(range(1000), filter(lambda x: x % 2 == 0), map(lambda x: x ** 2), list)
```

**When toolz wins:**
- Complex multi-step pipelines where readability matters
- Operations on iterators too large to fit in memory (lazy evaluation)
- When you need to reuse composed transformations
- When `cytoolz` is available and function dispatch overhead dominates

**When list comprehensions win:**
- Simple filter + map patterns
- Small data where overhead is proportionally large
- When the lambda syntax is more verbose than inline expressions

### 5. `toolz.curried` vs Regular `toolz`

A common source of confusion: `map` and `filter` from `toolz.curried` are curried, but the regular `toolz` namespace re-exports Python's built-in `map` and `filter` which are **not** curried.

```python
# This does NOT give you curried map/filter:
from toolz import map, filter  # these are just builtins

# This DOES:
from toolz.curried import map, filter

double = map(lambda x: x * 2)  # returns a curried function
double([1, 2, 3])               # <map object> -- still lazy!
list(double([1, 2, 3]))         # [2, 4, 6]
```

## Complete Code Examples

### Example 1: Data Pipeline with `pipe`

Process a log file to find the top 10 most common error messages:

```python
from toolz import pipe, frequencies, topk, take
from toolz.curried import map, filter

log_lines = [
    "2024-01-15 ERROR: Connection timeout",
    "2024-01-15 INFO: Request completed",
    "2024-01-15 ERROR: Connection timeout",
    "2024-01-15 ERROR: Invalid token",
    "2024-01-15 INFO: Request completed",
    "2024-01-15 ERROR: Connection timeout",
    "2024-01-15 WARN: Slow query",
    "2024-01-15 ERROR: Invalid token",
    "2024-01-15 ERROR: Disk full",
]

top_errors = pipe(
    log_lines,
    filter(lambda line: "ERROR" in line),       # keep only errors
    map(lambda line: line.split("ERROR: ")[1]),  # extract message
    frequencies,                                  # count occurrences
    lambda freq: topk(3, freq.items(), key=lambda x: x[1]),  # top 3
)

for msg, count in top_errors:
    print(f"  {msg}: {count}")
# Connection timeout: 3
# Invalid token: 2
# Disk full: 1
```

### Example 2: Curried Function Composition

Build reusable text processing pipelines:

```python
from toolz import curry, compose_left, pipe

@curry
def replace_chars(old, new, text):
    return text.replace(old, new)

@curry
def truncate(max_len, text):
    return text[:max_len] + "..." if len(text) > max_len else text

@curry
def ensure_prefix(prefix, text):
    return text if text.startswith(prefix) else prefix + text

# Build a reusable slug generator
slugify = compose_left(
    str.lower,
    str.strip,
    replace_chars(" ", "-"),
    replace_chars("_", "-"),
    truncate(50),
)

slugify("  Hello World  ")      # "hello-world"
slugify("My_Cool_Article  ")    # "my-cool-article"

# Build a URL path generator by extending the slug pipeline
make_url_path = compose_left(
    slugify,
    ensure_prefix("/blog/"),
)

make_url_path("My Article Title")  # "/blog/my-article-title"
```

### Example 3: Dictionary Transformations

Clean and transform configuration data:

```python
from toolz import pipe, merge, merge_with
from toolz.curried import keymap, valmap, keyfilter, valfilter, assoc

raw_config = {
    "DB_HOST": "localhost",
    "DB_PORT": "5432",
    "DB_NAME": "myapp",
    "DB_PASSWORD": "secret123",
    "APP_DEBUG": "true",
    "APP_PORT": "8080",
    "DEPRECATED_KEY": "old_value",
    "EMPTY_VAL": "",
}

# Extract DB config, clean keys, convert types
db_config = pipe(
    raw_config,
    keyfilter(lambda k: k.startswith("DB_")),         # only DB_ keys
    keymap(lambda k: k.replace("DB_", "").lower()),    # strip prefix, lowercase
    valfilter(bool),                                    # remove empty values
    lambda d: assoc(d, "port", int(d["port"])),        # convert port to int
)
# {'host': 'localhost', 'port': 5432, 'name': 'myapp', 'password': 'secret123'}

# Merge configs with conflict resolution
defaults = {"host": "0.0.0.0", "port": 3000, "debug": False}
overrides = {"port": 8080, "debug": True}

config = merge(defaults, overrides)
# {'host': '0.0.0.0', 'port': 8080, 'debug': True}

# Merge with custom conflict resolution
inventory = [
    {"apples": 5, "bananas": 3},
    {"apples": 2, "cherries": 10},
    {"bananas": 4, "cherries": 5},
]

totals = merge_with(sum, *inventory)
# {'apples': 7, 'bananas': 7, 'cherries': 15}
```

### Example 4: Grouping and Aggregation

Analyze a dataset of transactions:

```python
from toolz import pipe, groupby, valmap, reduceby, pluck, frequencies

transactions = [
    {"id": 1, "user": "alice", "amount": 50.0, "category": "food"},
    {"id": 2, "user": "bob",   "amount": 120.0, "category": "electronics"},
    {"id": 3, "user": "alice", "amount": 30.0, "category": "food"},
    {"id": 4, "user": "carol", "amount": 200.0, "category": "electronics"},
    {"id": 5, "user": "alice", "amount": 15.0, "category": "transport"},
    {"id": 6, "user": "bob",   "amount": 45.0, "category": "food"},
    {"id": 7, "user": "carol", "amount": 60.0, "category": "food"},
]

# Total spend per user using reduceby (single pass, memory efficient)
spend_per_user = reduceby(
    lambda t: t["user"],                  # key function
    lambda acc, t: acc + t["amount"],     # binary reduction
    transactions,
    init=0.0,
)
# {'alice': 95.0, 'bob': 165.0, 'carol': 260.0}

# Transaction count per category
category_counts = pipe(
    transactions,
    pluck("category"),
    frequencies,
)
# {'food': 4, 'electronics': 2, 'transport': 1}

# Average transaction amount per user
avg_per_user = pipe(
    transactions,
    groupby("user"),
    valmap(lambda txns: [t["amount"] for t in txns]),
    valmap(lambda amounts: round(sum(amounts) / len(amounts), 2)),
)
# {'alice': 31.67, 'bob': 82.5, 'carol': 130.0}

# Most expensive transaction per category
max_per_category = pipe(
    transactions,
    groupby("category"),
    valmap(lambda txns: max(txns, key=lambda t: t["amount"])),
    valmap(lambda t: {"user": t["user"], "amount": t["amount"]}),
)
# {'food': {'user': 'carol', 'amount': 60.0},
#  'electronics': {'user': 'carol', 'amount': 200.0},
#  'transport': {'user': 'alice', 'amount': 15.0}}
```

### Example 5: Functional Error Handling with `excepts`

Build a robust data ingestion pipeline that handles malformed records gracefully:

```python
from toolz import pipe, excepts, curry, count
from toolz.curried import map, filter

# Raw data with various problems
raw_records = [
    '{"name": "Alice", "age": "30", "score": "95.5"}',
    '{"name": "Bob", "age": "not_a_number", "score": "88.0"}',
    'not valid json at all',
    '{"name": "Carol", "age": "25", "score": ""}',
    '{"name": "Dave", "age": "40", "score": "72.3"}',
    None,
    '{"name": "Eve", "age": "35", "score": "91.0"}',
]

import json

# Build safe parsers with excepts
safe_json = excepts((json.JSONDecodeError, TypeError), json.loads, lambda e: None)

@curry
def safe_get(key, convert, record):
    """Safely extract and convert a field from a dict."""
    try:
        value = record[key]
        return convert(value)
    except (KeyError, ValueError, TypeError):
        return None

safe_int = safe_get(convert=int)
safe_float = safe_get(convert=float)

def parse_record(raw):
    """Parse a raw JSON string into a clean record, or None on failure."""
    parsed = safe_json(raw)
    if parsed is None:
        return None

    name = parsed.get("name")
    age = safe_int("age", parsed)
    score = safe_float("score", parsed)

    # Require all fields to be valid
    if any(v is None for v in [name, age, score]):
        return None

    return {"name": name, "age": age, "score": score}

# Process the pipeline
clean_records = pipe(
    raw_records,
    map(parse_record),
    filter(lambda r: r is not None),
    list,
)

for r in clean_records:
    print(r)
# {'name': 'Alice', 'age': 30, 'score': 95.5}
# {'name': 'Dave', 'age': 40, 'score': 72.3}
# {'name': 'Eve', 'age': 35, 'score': 91.0}

# We gracefully dropped 4 bad records:
#   - Bob: invalid age
#   - row 3: invalid JSON
#   - Carol: empty score
#   - row 6: None input
```

## Summary

`toolz` brings battle-tested functional programming primitives to Python without the weight of a framework. Its three sub-modules cover the core needs:

- **itertoolz** -- lazy sequence operations (the functional equivalent of shell pipes)
- **functoolz** -- function composition, currying, memoization (plumbing for building pipelines)
- **dicttoolz** -- immutable dictionary transforms (the functional way to work with Python's most common data structure)

The library's design philosophy is simple: small pure functions that compose. No classes, no decorators-on-decorators, no framework lock-in. Use what you need, ignore the rest. Swap in `cytoolz` for free performance when it matters.
