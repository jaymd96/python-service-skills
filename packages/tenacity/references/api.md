# tenacity — API Reference

> Part of the tenacity skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [The @retry Decorator](#the-retry-decorator)
  - [Full Parameter Reference](#full-parameter-reference)
- [Stop Strategies](#stop-strategies)
  - [stop_after_attempt](#stop_after_attemptmax_attempt_number)
  - [stop_after_delay](#stop_after_delaymax_delay)
  - [stop_never](#stop_never)
  - [stop_any](#stop_anystops)
  - [stop_all](#stop_allstops)
  - [stop_when_event_set](#stop_when_event_setevent)
- [Wait Strategies](#wait-strategies)
  - [wait_none](#wait_none)
  - [wait_fixed](#wait_fixedwait)
  - [wait_random](#wait_randommin-max)
  - [wait_exponential](#wait_exponentialmultiplier1-min0-maxinf-exp_base2)
  - [wait_exponential_jitter](#wait_exponential_jitterinitial1-max60-exp_base2-jitter1)
  - [wait_random_exponential](#wait_random_exponentialmultiplier1-maxinf-exp_base2-min0)
  - [wait_incrementing](#wait_incrementingstart0-increment100-maxinf)
  - [wait_combine](#wait_combinestrategies)
  - [wait_chain](#wait_chainstrategies)
- [Retry Conditions](#retry-conditions)
  - [retry_if_exception_type](#retry_if_exception_typeexception_types)
  - [retry_if_not_exception_type](#retry_if_not_exception_typeexception_types)
  - [retry_if_exception_message](#retry_if_exception_messagematchnone-messagenone)
  - [retry_if_exception_cause_type](#retry_if_exception_cause_typeexception_types)
  - [retry_if_result](#retry_if_resultpredicate)
  - [retry_if_not_result](#retry_if_not_resultpredicate)
  - [retry_if_exception](#retry_if_exceptionpredicate)
  - [retry_any and retry_all](#retry_anyconditions-and-retry_allconditions)
  - [retry_always and retry_never](#retry_always-and-retry_never)
- [Composability Patterns](#composability-patterns)
  - [Combining Stop Strategies](#combining-stop-strategies)
  - [Combining Wait Strategies](#combining-wait-strategies)
  - [Combining Retry Conditions](#combining-retry-conditions)
  - [Complex Compositions](#complex-compositions)
- [retry_with -- Modifying Retry Parameters](#retry_with----modifying-retry-parameters)
- [Context Manager Usage](#context-manager-usage)
  - [Accessing the Result with retry_state](#accessing-the-result-with-retry_state)
  - [Reraising Exceptions](#reraising-exceptions)
- [Summary Reference Table](#summary-reference-table)

---

### The `@retry` Decorator

The `@retry` decorator is the primary interface to tenacity. Applied to a function, it causes the function to be retried according to the configured strategies when it raises an exception.

```python
from tenacity import retry

@retry
def might_fail():
    """Will retry forever on any exception with no wait between attempts."""
    ...
```

#### Full Parameter Reference

```python
from tenacity import retry

@retry(
    # When to stop retrying
    stop=...,               # Stop strategy (e.g., stop_after_attempt(3))

    # How long to wait between retries
    wait=...,               # Wait strategy (e.g., wait_exponential())

    # Which exceptions/results trigger a retry
    retry=...,              # Retry condition (e.g., retry_if_exception_type(IOError))

    # Callbacks at various lifecycle points
    before=...,             # Called before each attempt: fn(retry_state)
    after=...,              # Called after each attempt: fn(retry_state)
    before_sleep=...,       # Called before sleeping between retries: fn(retry_state)

    # What to do when retries are exhausted
    retry_error_callback=...,  # Called when retries exhausted: fn(retry_state) -> value
    reraise=False,             # If True, reraise the last exception instead of RetryError

    # Wrapping behavior
    wrap_exception=False,      # If True, wrap non-retryable exceptions in RetryError

    # Retry state management
    retry_error_cls=RetryError,  # Custom RetryError class
)
def my_function():
    ...
```

**Important:** When `@retry` is used with no arguments (bare decorator), it retries **forever** on **any exception** with **no wait**. This is rarely what you want in production.

---

### Stop Strategies

Stop strategies determine when to give up retrying. Import from `tenacity`.

```python
from tenacity import (
    stop_after_attempt,
    stop_after_delay,
    stop_any,
    stop_all,
    stop_never,
    stop_when_event_set,
)
```

#### `stop_after_attempt(max_attempt_number)`

Stop after a fixed number of attempts.

```python
from tenacity import retry, stop_after_attempt

@retry(stop=stop_after_attempt(3))
def fetch_data():
    """Tries up to 3 times, then raises RetryError."""
    ...
```

- `max_attempt_number=3` means the function runs at most 3 times (1 initial + 2 retries).

#### `stop_after_delay(max_delay)`

Stop after a maximum total elapsed time (in seconds) since the first attempt.

```python
from tenacity import retry, stop_after_delay

@retry(stop=stop_after_delay(30))
def fetch_data():
    """Keeps retrying for up to 30 seconds total."""
    ...
```

- The timer starts at the first call, not the first retry.
- Accepts `int` or `float` for sub-second precision.

#### `stop_never`

Never stop retrying. This is the **default** when no `stop` is specified.

```python
from tenacity import retry, stop_never

@retry(stop=stop_never)  # Same as @retry with no stop
def fetch_data():
    ...
```

#### `stop_any(*stops)`

Stop when **any** of the given stop conditions is met (logical OR). Equivalent to combining with `|`.

```python
from tenacity import retry, stop_any, stop_after_attempt, stop_after_delay

# Stop after 5 attempts OR after 30 seconds, whichever comes first
@retry(stop=stop_any(stop_after_attempt(5), stop_after_delay(30)))
def fetch_data():
    ...

# Equivalent using the | operator:
@retry(stop=stop_after_attempt(5) | stop_after_delay(30))
def fetch_data():
    ...
```

#### `stop_all(*stops)`

Stop only when **all** of the given stop conditions are met (logical AND). Equivalent to combining with `&`.

```python
from tenacity import retry, stop_all, stop_after_attempt, stop_after_delay

# Stop only when BOTH conditions are true
@retry(stop=stop_all(stop_after_attempt(5), stop_after_delay(30)))
def fetch_data():
    ...

# Equivalent using the & operator:
@retry(stop=stop_after_attempt(5) & stop_after_delay(30))
def fetch_data():
    ...
```

#### `stop_when_event_set(event)`

Stop retrying when a `threading.Event` is set. Useful for graceful shutdown.

```python
import threading
from tenacity import retry, stop_when_event_set

shutdown_event = threading.Event()

@retry(stop=stop_when_event_set(shutdown_event))
def long_running_poll():
    ...

# In another thread:
shutdown_event.set()  # Causes the retry loop to stop
```

Also works with `asyncio.Event` for async functions.

---

### Wait Strategies

Wait strategies control the delay between retry attempts. Import from `tenacity`.

```python
from tenacity import (
    wait_fixed,
    wait_random,
    wait_exponential,
    wait_combine,
    wait_chain,
    wait_random_exponential,
    wait_incrementing,
    wait_exponential_jitter,
    wait_none,
)
```

#### `wait_none()`

No wait between retries (0 seconds). This is the **default**.

```python
from tenacity import retry, wait_none

@retry(wait=wait_none())
def fetch_data():
    ...
```

#### `wait_fixed(wait)`

Wait a fixed number of seconds between each retry.

```python
from tenacity import retry, wait_fixed

@retry(wait=wait_fixed(2))
def fetch_data():
    """Waits exactly 2 seconds between each retry."""
    ...
```

#### `wait_random(min, max)`

Wait a random amount of time between `min` and `max` seconds.

```python
from tenacity import retry, wait_random

@retry(wait=wait_random(min=1, max=5))
def fetch_data():
    """Waits between 1 and 5 seconds (random uniform)."""
    ...
```

#### `wait_exponential(multiplier=1, min=0, max=inf, exp_base=2)`

Exponential backoff. The wait time is `multiplier * exp_base ** (attempt - 1)`, clamped to `[min, max]`.

```python
from tenacity import retry, wait_exponential

@retry(wait=wait_exponential(multiplier=1, min=1, max=60))
def fetch_data():
    """
    Attempt 1: wait 1s (clamped from multiplier*2^0 = 1)
    Attempt 2: wait 2s (multiplier*2^1)
    Attempt 3: wait 4s (multiplier*2^2)
    Attempt 4: wait 8s (multiplier*2^3)
    ...capped at 60s
    """
    ...
```

Parameters:
- `multiplier` -- Multiplied with the exponential value. Default `1`.
- `min` -- Minimum wait in seconds. Default `0`.
- `max` -- Maximum wait in seconds. Default `inf` (math.inf).
- `exp_base` -- Base of the exponent. Default `2`.

#### `wait_exponential_jitter(initial=1, max=60, exp_base=2, jitter=1)`

Exponential backoff with added random jitter. The jitter helps avoid thundering herd problems when many clients retry simultaneously.

```python
from tenacity import retry, wait_exponential_jitter

@retry(wait=wait_exponential_jitter(initial=1, max=60, jitter=1))
def fetch_data():
    """
    Exponential backoff starting at 1s, max 60s,
    with up to 1s of random jitter added.
    """
    ...
```

Parameters:
- `initial` -- Initial wait time in seconds. Default `1`.
- `max` -- Maximum wait in seconds. Default `60`.
- `exp_base` -- Base of the exponent. Default `2`.
- `jitter` -- Maximum amount of random jitter to add (in seconds). Default `1`.

#### `wait_random_exponential(multiplier=1, max=inf, exp_base=2, min=0)`

Full jitter exponential backoff (also called "decorrelated jitter"). Wait time is `random(0, multiplier * exp_base ** (attempt - 1))`, clamped to `[min, max]`.

```python
from tenacity import retry, wait_random_exponential

@retry(wait=wait_random_exponential(multiplier=1, max=60))
def fetch_data():
    """Full jitter: random between 0 and the exponential ceiling."""
    ...
```

This is the strategy recommended by AWS for distributed systems (the "Full Jitter" algorithm).

#### `wait_incrementing(start=0, increment=100, max=inf)`

**Deprecated.** Wait time increases linearly: `start + increment * (attempt - 1)`.

```python
from tenacity import retry, wait_incrementing

@retry(wait=wait_incrementing(start=1, increment=2, max=30))
def fetch_data():
    """
    Attempt 1: 1s, Attempt 2: 3s, Attempt 3: 5s, ...capped at 30s
    """
    ...
```

> **Note:** `wait_incrementing` is deprecated. Use `wait_exponential` or a custom wait function instead.

#### `wait_combine(*strategies)`

Add multiple wait times together. The total wait is the sum of all strategies.

```python
from tenacity import retry, wait_combine, wait_fixed, wait_random

@retry(wait=wait_combine(wait_fixed(1), wait_random(0, 2)))
def fetch_data():
    """Waits 1s + random(0, 2)s = between 1s and 3s."""
    ...
```

Equivalent to using the `+` operator:

```python
@retry(wait=wait_fixed(1) + wait_random(0, 2))
def fetch_data():
    ...
```

#### `wait_chain(*strategies)`

Use different wait strategies sequentially. Each strategy is used for one attempt, then the next strategy takes over. The last strategy is used for all remaining attempts.

```python
from tenacity import retry, wait_chain, wait_fixed, wait_exponential

@retry(wait=wait_chain(
    wait_fixed(1),          # First retry: wait 1s
    wait_fixed(2),          # Second retry: wait 2s
    wait_fixed(5),          # Third retry: wait 5s
    wait_exponential(),     # All subsequent retries: exponential backoff
))
def fetch_data():
    ...
```

A common pattern is to use `wait_chain` with multiple `wait_fixed` for a stepped backoff:

```python
from tenacity import retry, wait_chain, wait_none, wait_fixed

@retry(wait=wait_chain(
    *[wait_none()] * 3,      # First 3 retries: no wait
    *[wait_fixed(1)] * 3,    # Next 3 retries: 1s wait
    *[wait_fixed(5)] * 3,    # Next 3 retries: 5s wait
))
def fetch_data():
    """Stepped backoff: 0, 0, 0, 1, 1, 1, 5, 5, 5, 5, 5, ..."""
    ...
```

---

### Retry Conditions

Retry conditions determine **which outcomes** (exceptions or return values) trigger a retry. By default, tenacity retries on any exception and never retries based on return values.

```python
from tenacity import (
    retry_if_exception_type,
    retry_if_not_exception_type,
    retry_if_exception_message,
    retry_if_exception_cause_type,
    retry_if_result,
    retry_if_not_result,
    retry_any,
    retry_all,
    retry_always,
    retry_never,
    retry_if_exception,
    retry_unless_exception_type,
)
```

#### `retry_if_exception_type(exception_types)`

Retry only when specific exception types are raised.

```python
from tenacity import retry, retry_if_exception_type

@retry(retry=retry_if_exception_type(IOError))
def read_file():
    """Only retries on IOError, other exceptions propagate immediately."""
    ...

# Multiple exception types:
@retry(retry=retry_if_exception_type((IOError, ConnectionError, TimeoutError)))
def fetch_data():
    ...
```

#### `retry_if_not_exception_type(exception_types)`

Retry on any exception **except** the specified types. The specified exception types propagate immediately.

```python
from tenacity import retry, retry_if_not_exception_type

@retry(retry=retry_if_not_exception_type(ValueError))
def process():
    """Retries on anything except ValueError."""
    ...
```

#### `retry_if_exception_message(match=None, message=None)`

Retry when the exception message matches a pattern.

```python
from tenacity import retry, retry_if_exception_message

# Exact substring match:
@retry(retry=retry_if_exception_message(message="temporarily unavailable"))
def fetch():
    ...

# Regex match:
@retry(retry=retry_if_exception_message(match=r".*timeout.*"))
def fetch():
    ...
```

Parameters:
- `message` -- Substring to search for in the exception message.
- `match` -- Regular expression pattern to match against the exception message.

#### `retry_if_exception_cause_type(exception_types)`

Retry when the exception's `__cause__` (from `raise ... from ...`) matches the specified type.

```python
from tenacity import retry, retry_if_exception_cause_type

@retry(retry=retry_if_exception_cause_type(ConnectionError))
def fetch():
    ...
```

#### `retry_if_result(predicate)`

Retry based on the function's **return value**. The predicate receives the return value and should return `True` to trigger a retry.

```python
from tenacity import retry, retry_if_result

@retry(retry=retry_if_result(lambda result: result is None))
def get_value():
    """Retries if the function returns None."""
    ...

@retry(retry=retry_if_result(lambda r: r.status_code >= 500))
def call_api():
    """Retries on 5xx responses."""
    ...
```

#### `retry_if_not_result(predicate)`

Retry when the return value does **not** match the predicate. Retries if the predicate returns `False`.

```python
from tenacity import retry, retry_if_not_result

@retry(retry=retry_if_not_result(lambda r: r == "success"))
def do_work():
    """Retries until the function returns 'success'."""
    ...
```

#### `retry_if_exception(predicate)`

Retry when an exception is raised and the predicate function returns `True` for that exception.

```python
from tenacity import retry, retry_if_exception

@retry(retry=retry_if_exception(lambda e: hasattr(e, 'retryable') and e.retryable))
def call_service():
    """Retries only if the exception has a truthy .retryable attribute."""
    ...
```

#### `retry_any(*conditions)` and `retry_all(*conditions)`

Combine multiple retry conditions with OR (`retry_any`) or AND (`retry_all`) logic.

```python
from tenacity import (
    retry, retry_any, retry_all,
    retry_if_exception_type, retry_if_result,
)

# Retry on IOError OR if result is None (OR logic):
@retry(retry=retry_any(
    retry_if_exception_type(IOError),
    retry_if_result(lambda r: r is None),
))
def fetch():
    ...

# Equivalent using the | operator:
@retry(retry=retry_if_exception_type(IOError) | retry_if_result(lambda r: r is None))
def fetch():
    ...

# AND logic (both conditions must be true):
@retry(retry=retry_all(
    retry_if_exception_type(IOError),
    retry_if_exception_message(match=r".*transient.*"),
))
def fetch():
    ...

# Equivalent using the & operator:
@retry(retry=(
    retry_if_exception_type(IOError) & retry_if_exception_message(match=r".*transient.*")
))
def fetch():
    ...
```

#### `retry_always` and `retry_never`

Constants for always retrying or never retrying (used internally and for testing).

```python
from tenacity import retry_always, retry_never
```

---

## Composability Patterns

One of tenacity's greatest strengths is the ability to combine strategies using Python's `|` (OR) and `&` (AND) operators.

### Combining Stop Strategies

```python
from tenacity import retry, stop_after_attempt, stop_after_delay

# Stop after 5 attempts OR 30 seconds (whichever comes first):
@retry(stop=stop_after_attempt(5) | stop_after_delay(30))
def fetch():
    ...

# Stop only when BOTH are true (at least 3 attempts AND at least 10 seconds):
@retry(stop=stop_after_attempt(3) & stop_after_delay(10))
def fetch():
    ...
```

### Combining Wait Strategies

```python
from tenacity import retry, wait_fixed, wait_random

# Add wait times together (1s fixed + 0-2s random = 1-3s total):
@retry(wait=wait_fixed(1) + wait_random(0, 2))
def fetch():
    ...
```

### Combining Retry Conditions

```python
from tenacity import retry, retry_if_exception_type, retry_if_result

# Retry on ConnectionError OR if result is None:
@retry(retry=retry_if_exception_type(ConnectionError) | retry_if_result(lambda r: r is None))
def fetch():
    ...

# Retry only if it's an IOError AND the message contains "transient":
@retry(retry=(
    retry_if_exception_type(IOError)
    & retry_if_exception_message(match=r"transient")
))
def fetch():
    ...
```

### Complex Compositions

```python
from tenacity import (
    retry, stop_after_attempt, stop_after_delay,
    wait_exponential, wait_random,
    retry_if_exception_type, retry_if_result,
    before_sleep_log,
)
import logging

logger = logging.getLogger(__name__)

@retry(
    # Stop after 10 attempts or 2 minutes
    stop=stop_after_attempt(10) | stop_after_delay(120),
    # Exponential backoff with jitter
    wait=wait_exponential(multiplier=1, min=1, max=30) + wait_random(0, 2),
    # Retry on network errors or 5xx responses
    retry=(
        retry_if_exception_type((ConnectionError, TimeoutError))
        | retry_if_result(lambda r: r.status_code >= 500)
    ),
    # Log before sleeping
    before_sleep=before_sleep_log(logger, logging.WARNING),
    # Reraise the original exception instead of RetryError
    reraise=True,
)
def resilient_api_call(url):
    ...
```

---

### `retry_with()` -- Modifying Retry Parameters

Use `.retry_with()` to create a copy of a retry-decorated function with modified parameters. The original function is not affected.

```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))
def fetch_data():
    ...

# Create a variant with more retries and longer wait:
fetch_data_patient = fetch_data.retry_with(
    stop=stop_after_attempt(10),
    wait=wait_fixed(5),
)

# Original function still uses 3 attempts / 1s wait
fetch_data()

# New variant uses 10 attempts / 5s wait
fetch_data_patient()
```

This is especially useful for testing, where you want to reduce retry counts:

```python
def test_fetch_data():
    fast_fetch = fetch_data.retry_with(
        stop=stop_after_attempt(1),
        wait=wait_fixed(0),
    )
    with pytest.raises(RetryError):
        fast_fetch()
```

---

### Context Manager Usage

Tenacity can be used as a context manager via the `Retrying` class, which is useful when you need retry logic around a block of code rather than a single function.

```python
from tenacity import Retrying, stop_after_attempt, wait_fixed

for attempt in Retrying(stop=stop_after_attempt(3), wait=wait_fixed(1)):
    with attempt:
        result = some_operation()
```

The context manager approach:
- Iterates over retry attempts
- Each `attempt` is a context manager that captures exceptions
- If the block raises, tenacity decides whether to retry
- If it succeeds, the loop ends naturally
- `result` is available after a successful attempt

#### Accessing the Result with `retry_state`

```python
from tenacity import Retrying, stop_after_attempt

for attempt in Retrying(stop=stop_after_attempt(3)):
    with attempt:
        value = compute_something()

# After the loop, you can also check attempt state
print(f"Succeeded after {attempt.retry_state.attempt_number} attempts")
```

#### Reraising Exceptions

By default the context manager wraps exceptions in `RetryError`. Use `reraise=True` to get the original exception:

```python
from tenacity import Retrying, stop_after_attempt

for attempt in Retrying(stop=stop_after_attempt(3), reraise=True):
    with attempt:
        do_something()
# If all 3 attempts fail, the original exception is raised (not RetryError)
```

---

## Summary Reference Table

### Stop Strategies

| Strategy | Description |
|----------|-------------|
| `stop_never` | Never stop (default) |
| `stop_after_attempt(n)` | Stop after `n` total attempts |
| `stop_after_delay(seconds)` | Stop after total elapsed time |
| `stop_any(*stops)` / `s1 \| s2` | Stop when any condition is met |
| `stop_all(*stops)` / `s1 & s2` | Stop when all conditions are met |
| `stop_when_event_set(event)` | Stop when threading/asyncio Event is set |

### Wait Strategies

| Strategy | Description |
|----------|-------------|
| `wait_none()` | No wait (default) |
| `wait_fixed(seconds)` | Fixed delay |
| `wait_random(min, max)` | Uniform random delay |
| `wait_exponential(multiplier, min, max, exp_base)` | Exponential backoff |
| `wait_exponential_jitter(initial, max, exp_base, jitter)` | Exponential backoff + jitter |
| `wait_random_exponential(multiplier, max, exp_base)` | Full jitter backoff |
| `wait_incrementing(start, increment, max)` | Linear increase (deprecated) |
| `wait_combine(*strategies)` / `w1 + w2` | Sum of wait times |
| `wait_chain(*strategies)` | Sequential strategies |

### Retry Conditions

| Condition | Description |
|-----------|-------------|
| `retry_if_exception_type(exc)` | Retry on specific exception types |
| `retry_if_not_exception_type(exc)` | Retry on all exceptions except specified |
| `retry_if_exception_message(match, message)` | Retry when exception message matches |
| `retry_if_exception_cause_type(exc)` | Retry when exception `__cause__` matches |
| `retry_if_exception(predicate)` | Retry when predicate(exception) is True |
| `retry_if_result(predicate)` | Retry when predicate(result) is True |
| `retry_if_not_result(predicate)` | Retry when predicate(result) is False |
| `retry_any(*conds)` / `c1 \| c2` | Retry when any condition is met |
| `retry_all(*conds)` / `c1 & c2` | Retry when all conditions are met |
