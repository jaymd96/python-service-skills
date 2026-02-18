# toolz

Functional programming primitives for Python -- the **coreutils of FP**.

`toolz` provides a set of utility functions for working with iterables, functions, and dictionaries in a functional style. It extends Python's `itertools` and `functools` standard library modules with a consistent, composable API that favors pure functions, laziness, and pipeline-oriented data processing.

---

## Latest Stable Version

| Detail | Value |
|--------|-------|
| **Package** | `toolz` |
| **Latest version** | 1.0.0 (released July 2024) |
| **Python support** | Python >= 3.8 |
| **License** | BSD 3-Clause |
| **Repository** | https://github.com/pytoolz/toolz |
| **Documentation** | https://toolz.readthedocs.io |

The 1.0.0 release marked the package as stable after years at 0.x. The API has been effectively frozen since ~0.10 and is considered production-ready.

---

## Installation

```bash
# Standard installation
pip install toolz

# C-accelerated version (same API, faster execution)
pip install cytoolz
```

`toolz` has **zero dependencies**. It is pure Python and works everywhere CPython runs.

If you install `cytoolz`, you can use it as a drop-in replacement:

```python
# Option 1: always use cytoolz
from cytoolz import pipe, curry, groupby

# Option 2: fall back gracefully
try:
    from cytoolz import pipe, curry, groupby
except ImportError:
    from toolz import pipe, curry, groupby

# Option 3: use toolz.curried for pre-curried versions
from toolz.curried import map, filter, reduce, get
```

---

## Core API by Category

### Itertoolz

Functions for working with iterables. Most return **lazy iterators** (not lists).

| Function | Signature | Description |
|----------|-----------|-------------|
| `accumulate` | `accumulate(binop, seq, initial='__no__default__')` | Running reduction. Like `itertools.accumulate` but takes the function first. |
| `concat` | `concat(seqs)` | Concatenate zero or more iterables (lazy `itertools.chain.from_iterable`). |
| `concatv` | `concatv(*seqs)` | Variadic version of `concat`. `concatv([1], [2])` instead of `concat([[1], [2]])`. |
| `cons` | `cons(el, seq)` | Prepend element to an iterable. `cons(1, [2, 3])` -> `(1, 2, 3)`. |
| `count` | `count(seq)` | Count elements in an iterable (consumes it). |
| `drop` | `drop(n, seq)` | Drop the first `n` elements from a sequence. |
| `first` | `first(seq)` | Return the first element. |
| `frequencies` | `frequencies(seq)` | Count occurrences of each element. Returns a dict. |
| `get` | `get(ind, seq, default='__no__default__')` | Get element(s) by index. Supports multiple indices: `get([0, 2], 'abcde')` -> `('a', 'c')`. |
| `groupby` | `groupby(key, seq)` | Group elements by a key function. Returns `{key: [elements]}`. |
| `interleave` | `interleave(seqs)` | Interleave elements from multiple iterables. `interleave([[1,2],[3,4]])` -> `(1,3,2,4)`. |
| `interpose` | `interpose(el, seq)` | Insert element between each pair. `interpose('x', [1,2,3])` -> `(1,'x',2,'x',3)`. |
| `isdistinct` | `isdistinct(seq)` | Return True if all elements are unique. |
| `isiterable` | `isiterable(x)` | Return True if `x` is iterable (but not a string). |
| `iterate` | `iterate(func, x)` | Infinite sequence: `x, func(x), func(func(x)), ...` |
| `join` | `join(leftkey, leftseq, rightkey, rightseq, left_default='__no__default__', right_default='__no__default__')` | Join two sequences like a SQL join on key functions. |
| `last` | `last(seq)` | Return the last element. |
| `mapcat` | `mapcat(func, seqs)` | Map then concatenate. `mapcat(f, seqs)` == `concat(map(f, seqs))`. |
| `merge_sorted` | `merge_sorted(*seqs, key=None)` | Merge pre-sorted iterables into a single sorted iterable. |
| `nth` | `nth(n, seq)` | Return the nth element (0-indexed). |
| `partition` | `partition(n, seq, pad='__no__default__')` | Split into tuples of length `n`. Drops incomplete tail unless `pad` given. |
| `partition_all` | `partition_all(n, seq)` | Like `partition` but includes the incomplete final tuple. |
| `peek` | `peek(seq)` | Retrieve the first element and return `(first, iterator_with_first_restored)`. |
| `pluck` | `pluck(ind, seqs, default='__no__default__')` | Extract field(s) from each element. `pluck('name', records)`. |
| `random_sample` | `random_sample(prob, seq, random_state=None)` | Lazily sample elements with probability `prob`. |
| `reduceby` | `reduceby(key, binop, seq, init='__no__default__')` | Grouped reduction in a single pass. `reduceby(keyfunc, add, data)`. |
| `remove` | `remove(predicate, seq)` | Remove elements where predicate is True (opposite of `filter`). |
| `second` | `second(seq)` | Return the second element. |
| `sliding_window` | `sliding_window(n, seq)` | Sliding window of size `n`. `sliding_window(2, [1,2,3,4])` -> `(1,2),(2,3),(3,4)`. |
| `tail` | `tail(n, seq)` | Return the last `n` elements. |
| `take` | `take(n, seq)` | Return the first `n` elements as a list. |
| `take_nth` | `take_nth(n, seq)` | Take every nth element. `take_nth(2, [1,2,3,4,5])` -> `(1,3,5)`. |
| `topk` | `topk(k, seq, key=None)` | Return the `k` largest elements. Uses a heap, O(n log k). |
| `unique` | `unique(seq, key=None)` | Return unique elements, preserving order. |

### Functoolz

Higher-order functions for composing, currying, and transforming functions.

| Function | Signature | Description |
|----------|-----------|-------------|
| `apply` | `apply(*func_and_args, **kwargs)` | Call a function with positional and keyword arguments. `apply(f, x)` == `f(x)`. |
| `complement` | `complement(func)` | Logical negation of a predicate. `complement(iseven)(3)` -> `True`. |
| `compose` | `compose(*funcs)` | Right-to-left function composition. `compose(f, g)(x)` == `f(g(x))`. |
| `compose_left` | `compose_left(*funcs)` | Left-to-right function composition. `compose_left(f, g)(x)` == `g(f(x))`. |
| `curry` | `curry(func, *args, **kwargs)` | Automatic partial application. Returns a new function until all args are provided. |
| `do` | `do(func, x)` | Call `func(x)` for side effect, then return `x`. For inserting logging into pipelines. |
| `excepts` | `excepts(exc, func, handler=None)` | Catch exceptions functionally. Returns `handler(e)` on exception, `func(x)` otherwise. |
| `flip` | `flip(func)` | Reverse the argument order. `flip(func)(a, b)` == `func(b, a)`. |
| `identity` | `identity(x)` | Return `x` unchanged. Useful as default/no-op in pipelines. |
| `juxt` | `juxt(*funcs)` | Apply multiple functions to the same arguments. `juxt(f, g)(x)` -> `(f(x), g(x))`. |
| `memoize` | `memoize(func, cache=None, key=None)` | Cache function results. Supports custom cache backends. |
| `pipe` | `pipe(data, *funcs)` | Thread data through a sequence of functions. `pipe(x, f, g)` == `g(f(x))`. |
| `thread_first` | `thread_first(val, *forms)` | Thread value as the **first** argument through a sequence of function calls. |
| `thread_last` | `thread_last(val, *forms)` | Thread value as the **last** argument through a sequence of function calls. |

### Dicttoolz

Functions for working with dictionaries in a functional, non-mutating style.

| Function | Signature | Description |
|----------|-----------|-------------|
| `assoc` | `assoc(d, key, value, factory=dict)` | Return a new dict with `key` set to `value`. Does not mutate `d`. |
| `assoc_in` | `assoc_in(d, keys, value, factory=dict)` | Set a value in a nested dict. `assoc_in({}, ['a','b'], 1)` -> `{'a': {'b': 1}}`. |
| `dissoc` | `dissoc(d, *keys, factory=dict)` | Return a new dict without specified keys. |
| `get_in` | `get_in(keys, coll, default=None, no_default=False)` | Get a value from a nested dict. `get_in(['a','b'], {'a': {'b': 1}})` -> `1`. |
| `itemfilter` | `itemfilter(predicate, d, factory=dict)` | Filter dict by `(key, value)` pairs. |
| `itemmap` | `itemmap(func, d, factory=dict)` | Apply function to each `(key, value)` pair, returning new dict. |
| `keyfilter` | `keyfilter(predicate, d, factory=dict)` | Filter dict by keys. `keyfilter(lambda k: k != 'x', d)`. |
| `keymap` | `keymap(func, d, factory=dict)` | Apply function to each key. `keymap(str.upper, {'a': 1})` -> `{'A': 1}`. |
| `merge` | `merge(*dicts, **kwargs)` | Merge dictionaries left-to-right. Later values win. |
| `merge_with` | `merge_with(func, *dicts, **kwargs)` | Merge dicts, applying `func` to collisions. `merge_with(sum, d1, d2)`. |
| `update_in` | `update_in(d, keys, func, default=None, factory=dict)` | Update a value in a nested dict by applying a function. |
| `valfilter` | `valfilter(predicate, d, factory=dict)` | Filter dict by values. `valfilter(lambda v: v > 0, d)`. |
| `valmap` | `valmap(func, d, factory=dict)` | Apply function to each value. `valmap(len, {'a': [1,2], 'b': [3]})` -> `{'a': 2, 'b': 1}`. |

---

## Key Patterns

### `pipe()` for Data Pipeline Composition

`pipe` is the workhorse of toolz. It threads a value through a sequence of functions left-to-right, making data transformations read top-to-bottom like shell pipelines.

```python
from toolz import pipe, curry
from toolz.curried import map, filter

result = pipe(
    range(100),
    filter(lambda x: x % 2 == 0),   # keep evens
    map(lambda x: x ** 2),           # square them
    sum,                              # add them up
)
# result: 161700
```

The key insight: use `toolz.curried` for pre-curried versions of builtins that accept the iterable last, making them pipeline-friendly.

### `curry()` for Partial Application

`curry` automatically returns a new function when called with fewer arguments than expected. Unlike `functools.partial`, it chains automatically:

```python
from toolz import curry

@curry
def add(x, y):
    return x + y

@curry
def multiply(x, y):
    return x * y

add5 = add(5)
double = multiply(2)

add5(3)     # 8
double(10)  # 20

# Compose curried functions in pipelines
from toolz import pipe

pipe(10, add(5), multiply(3))  # (10 + 5) * 3 = 45
```

### `compose()` vs `compose_left()` vs `pipe()`

All three combine functions, but differ in argument order and usage:

```python
from toolz import compose, compose_left, pipe

def double(x): return x * 2
def inc(x): return x + 1
def square(x): return x ** 2

# compose: right-to-left (mathematical notation)
# Reads "square of inc of double of x"
f = compose(square, inc, double)
f(3)  # square(inc(double(3))) = square(inc(6)) = square(7) = 49

# compose_left: left-to-right (pipeline notation)
# Reads "double, then inc, then square"
g = compose_left(double, inc, square)
g(3)  # square(inc(double(3))) = 49  (same result!)

# pipe: like compose_left, but immediately applies to data
pipe(3, double, inc, square)  # 49
```

**When to use each:**
- `pipe(data, f, g, h)` -- you have data now and want to transform it
- `compose_left(f, g, h)` -- you want a reusable function, reading left-to-right
- `compose(h, g, f)` -- you want a reusable function, reading right-to-left (math convention)

### `groupby()` + `valmap()` for Data Transformation

This is the split-apply-combine pattern, done functionally:

```python
from toolz import groupby, valmap, pipe

records = [
    {"name": "Alice", "dept": "Engineering", "salary": 120_000},
    {"name": "Bob", "dept": "Engineering", "salary": 110_000},
    {"name": "Carol", "dept": "Marketing", "salary": 95_000},
    {"name": "Dave", "dept": "Marketing", "salary": 90_000},
    {"name": "Eve", "dept": "Engineering", "salary": 130_000},
]

# Average salary by department
avg_by_dept = pipe(
    records,
    lambda rs: groupby("dept", rs),                        # split
    lambda groups: valmap(lambda g: [r["salary"] for r in g], groups),  # extract
    lambda salaries: valmap(lambda s: sum(s) / len(s), salaries),       # combine
)
# {'Engineering': 120000.0, 'Marketing': 92500.0}
```

A cleaner version using curried functions:

```python
from toolz import pipe, groupby, valmap, pluck

def mean(xs):
    xs = list(xs)
    return sum(xs) / len(xs)

avg_by_dept = pipe(
    records,
    groupby("dept"),
    valmap(lambda g: mean(pluck("salary", g))),
)
```

### `memoize()` for Caching

`memoize` caches function results based on arguments. Simpler than `functools.lru_cache` for the common case:

```python
from toolz import memoize

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)  # instant, no stack overflow thanks to bottom-up caching

# Custom cache backend (e.g., LRU with max size)
from functools import lru_cache

@memoize(cache={})
def expensive_lookup(key):
    ...

# Inspect or clear cache
expensive_lookup.cache  # the underlying dict
```

### `excepts()` for Functional Error Handling

`excepts` wraps a function to catch specific exceptions, returning a handler's result instead of raising:

```python
from toolz import excepts, pipe
from toolz.curried import map

safe_int = excepts(ValueError, int, lambda e: None)

safe_int("42")    # 42
safe_int("hello") # None
safe_int("")      # None

# Use in pipelines to handle bad data gracefully
raw = ["1", "2", "bad", "4", "oops", "6"]

pipe(
    raw,
    map(safe_int),         # [1, 2, None, 4, None, 6]
    filter(None.__ne__),   # [1, 2, 4, 6]
    list,
)
# [1, 2, 4, 6]
```

Handle multiple exception types:

```python
safe_divide = excepts(
    (ZeroDivisionError, TypeError),
    lambda x: 100 / x,
    lambda e: float("inf"),
)

safe_divide(5)    # 20.0
safe_divide(0)    # inf
safe_divide(None) # inf
```

---

## cytoolz -- Cython-Accelerated Version

`cytoolz` is a Cython implementation of toolz with an identical API. It provides significant speedups for hot paths.

### When to Use cytoolz

| Scenario | Recommendation |
|----------|----------------|
| General scripting | `toolz` is fine, zero compile overhead |
| Data pipelines with millions of records | `cytoolz` gives 2-5x speedup on function call overhead |
| Libraries consumed by others | Use `toolz`, let consumers choose `cytoolz` |
| Performance-critical inner loops | `cytoolz` -- the function call overhead reduction matters |

### Typical Speedups

- `curry` dispatch: ~3-5x faster
- `pipe` with many stages: ~2-4x faster
- `groupby`, `frequencies`, `reduceby`: ~2-3x faster
- `compose` call overhead: ~3-5x faster

### Installation and Usage

```bash
pip install cytoolz
```

```python
# Direct import
from cytoolz import pipe, curry, groupby

# Graceful fallback pattern (recommended for libraries)
try:
    from cytoolz import pipe, curry, groupby
    from cytoolz.curried import map, filter
except ImportError:
    from toolz import pipe, curry, groupby
    from toolz.curried import map, filter
```

`cytoolz` requires a C compiler to build from source, but binary wheels are available for most platforms. If you cannot install `cytoolz` (e.g., in constrained environments), `toolz` works identically, just slower on function dispatch overhead.

### cytoolz.curried

Like `toolz.curried`, `cytoolz.curried` provides pre-curried versions of all functions:

```python
from cytoolz.curried import get, pluck, groupby, valmap
```

---

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

---

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

---

## `toolz.curried` Module Reference

The `toolz.curried` namespace provides curried versions of all toolz functions, plus curried versions of key builtins. This is essential for building clean pipelines.

```python
import toolz.curried as tz

# All toolz functions are curried:
tz.map(str.upper)            # returns a curried function waiting for an iterable
tz.filter(bool)              # returns a curried function waiting for an iterable
tz.get(0)                    # returns a curried function waiting for a sequence
tz.pluck("name")             # returns a curried function waiting for a sequence of dicts
tz.groupby(len)              # returns a curried function waiting for a sequence
tz.keymap(str.upper)         # returns a curried function waiting for a dict
tz.valmap(len)               # returns a curried function waiting for a dict
tz.valfilter(bool)           # returns a curried function waiting for a dict

# Complete pipeline using only curried functions:
result = tz.pipe(
    [{"name": "Alice", "score": 90}, {"name": "Bob", "score": 85}],
    tz.pluck("score"),
    tz.map(lambda s: s * 1.1),
    sum,
)
```

---

## Summary

`toolz` brings battle-tested functional programming primitives to Python without the weight of a framework. Its three sub-modules cover the core needs:

- **itertoolz** -- lazy sequence operations (the functional equivalent of shell pipes)
- **functoolz** -- function composition, currying, memoization (plumbing for building pipelines)
- **dicttoolz** -- immutable dictionary transforms (the functional way to work with Python's most common data structure)

The library's design philosophy is simple: small pure functions that compose. No classes, no decorators-on-decorators, no framework lock-in. Use what you need, ignore the rest. Swap in `cytoolz` for free performance when it matters.
