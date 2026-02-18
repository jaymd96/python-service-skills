# Tenacity -- Comprehensive Python Retry Library

## Overview

**Tenacity** is a general-purpose Python retrying library that provides declarative retry behavior for any callable. Originally forked from the `retrying` library, tenacity is a complete rewrite that addresses design issues in its predecessor and adds a rich set of features for controlling retry logic.

**Key Characteristics:**

- **Version:** 9.1.2 (latest stable as of early 2026)
- **Python:** 3.9+
- **License:** Apache 2.0
- **Dependencies:** None (pure Python)
- **Thread Safety:** Yes (each decorated function gets independent retry state)
- **Async Support:** Native asyncio and Tornado support

**When to use Tenacity:**

- Retrying failed HTTP/API requests with backoff
- Reconnecting to databases or message brokers after transient failures
- Polling until a condition is met (e.g., waiting for a resource to become ready)
- Retrying any operation that can fail due to transient, recoverable errors
- Wrapping flaky third-party calls with resilience logic

**Core Design Principles:**

- Composable -- stop, wait, and retry conditions combine with `|` and `&` operators
- Decorator-based -- minimal code intrusion via `@retry`
- Callback-driven -- hook into every stage of the retry lifecycle
- Async-native -- first-class asyncio support without wrappers

---

## Installation

```bash
pip install tenacity
```

For the latest version:

```bash
pip install tenacity>=9.0
```

Tenacity has **zero runtime dependencies**. It is pure Python.

---

## Core API

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

### Before/After Callbacks

Callbacks let you hook into the retry lifecycle for logging, metrics, or other side effects.

```python
from tenacity import (
    before_log,
    after_log,
    before_sleep_log,
    before_nothing,
    after_nothing,
)
```

#### `before_log(logger, log_level)`

Log a message **before** each attempt.

```python
import logging
from tenacity import retry, before_log, stop_after_attempt

logger = logging.getLogger(__name__)

@retry(stop=stop_after_attempt(3), before=before_log(logger, logging.DEBUG))
def fetch_data():
    ...
```

Logs output like:
```
Starting call to '__main__.fetch_data', this is the 1st time calling it.
Starting call to '__main__.fetch_data', this is the 2nd time calling it.
```

#### `after_log(logger, log_level)`

Log a message **after** each failed attempt (before deciding whether to retry).

```python
import logging
from tenacity import retry, after_log, stop_after_attempt

logger = logging.getLogger(__name__)

@retry(stop=stop_after_attempt(3), after=after_log(logger, logging.WARNING))
def fetch_data():
    ...
```

Logs output like:
```
Finished call to '__main__.fetch_data' after 0.001(s), this was the 1st time calling it.
```

#### `before_sleep_log(logger, log_level, exc_info=False)`

Log a message **before sleeping** between retries. This is the most informative callback since it includes the wait time and the exception that triggered the retry.

```python
import logging
from tenacity import retry, before_sleep_log, stop_after_attempt, wait_fixed

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    before_sleep=before_sleep_log(logger, logging.WARNING, exc_info=True),
)
def fetch_data():
    ...
```

Logs output like:
```
Retrying __main__.fetch_data in 2.0 seconds as it raised ConnectionError: refused.
```

Set `exc_info=True` to include the full traceback in the log entry.

#### Custom Callbacks

All callback parameters accept any callable with the signature `fn(retry_state: RetryCallState) -> None`.

```python
from tenacity import retry, stop_after_attempt

def my_before_callback(retry_state):
    print(f"Attempt #{retry_state.attempt_number} for {retry_state.fn.__name__}")

def my_after_callback(retry_state):
    if retry_state.outcome.failed:
        print(f"Attempt #{retry_state.attempt_number} failed: {retry_state.outcome.exception()}")
    else:
        print(f"Attempt #{retry_state.attempt_number} returned: {retry_state.outcome.result()}")

@retry(
    stop=stop_after_attempt(3),
    before=my_before_callback,
    after=my_after_callback,
)
def fetch_data():
    ...
```

---

### `RetryCallState`

The `RetryCallState` object is passed to all callbacks and custom stop/wait/retry functions. It contains all information about the current retry lifecycle.

#### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `attempt_number` | `int` | Current attempt number (starts at 1) |
| `outcome` | `Future` | The `concurrent.futures.Future` holding the result or exception of the last attempt |
| `outcome_timestamp` | `float` | Timestamp when the outcome was recorded |
| `start_time` | `float` | Monotonic timestamp when the first attempt started |
| `idle_for` | `float` | Total seconds spent sleeping (waiting) so far |
| `retry_object` | `Retrying` | The `Retrying` instance managing this retry loop |
| `fn` | `callable` | The wrapped function being retried |
| `args` | `tuple` | Positional arguments passed to the function |
| `kwargs` | `dict` | Keyword arguments passed to the function |
| `next_action` | `RetryAction` | The next action to take (sleep duration or stop) |

#### Accessing Outcome Data

```python
def my_callback(retry_state):
    if retry_state.outcome is None:
        # Before the first attempt, outcome is None
        return

    if retry_state.outcome.failed:
        # The attempt raised an exception
        exc = retry_state.outcome.exception()
        print(f"Exception: {exc}")
    else:
        # The attempt returned a value
        result = retry_state.outcome.result()
        print(f"Result: {result}")

    # Total elapsed time
    elapsed = retry_state.outcome_timestamp - retry_state.start_time
    print(f"Elapsed: {elapsed:.2f}s over {retry_state.attempt_number} attempts")
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

### Async Support

Tenacity natively supports `async` functions. The `@retry` decorator automatically detects async functions and handles them correctly.

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential())
async def async_fetch(url: str):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            if response.status >= 500:
                raise Exception(f"Server error: {response.status}")
            return await response.json()

# Usage:
result = await async_fetch("https://api.example.com/data")
```

#### Async Context Manager

For async code, use `AsyncRetrying`:

```python
from tenacity import AsyncRetrying, stop_after_attempt, wait_fixed

async def do_work():
    async for attempt in AsyncRetrying(stop=stop_after_attempt(3), wait=wait_fixed(1)):
        with attempt:
            result = await some_async_operation()
```

#### Async Event-Based Stop

Use `asyncio.Event` with `stop_when_event_set` for graceful async shutdown:

```python
import asyncio
from tenacity import retry, stop_when_event_set

shutdown = asyncio.Event()

@retry(stop=stop_when_event_set(shutdown))
async def poll_service():
    ...

# To stop:
shutdown.set()
```

#### Async Sleep Override

By default, async retries use `asyncio.sleep`. You can override the sleep function:

```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(1), sleep=asyncio.sleep)
async def my_async_fn():
    ...
```

---

### Statistics

Every retry-decorated function tracks statistics about its retry behavior via the `.statistics` attribute.

```python
from tenacity import retry, stop_after_attempt

@retry(stop=stop_after_attempt(3))
def fetch_data():
    ...

# After calling the function:
try:
    fetch_data()
except Exception:
    pass

print(fetch_data.statistics)
```

The `statistics` dictionary contains:

| Key | Type | Description |
|-----|------|-------------|
| `"start_time"` | `float` | Monotonic time when the first attempt started |
| `"attempt_number"` | `int` | Total number of attempts made |
| `"idle_for"` | `float` | Total time spent waiting (sleeping) between retries |

```python
stats = fetch_data.statistics
print(f"Attempts: {stats['attempt_number']}")
print(f"Total wait time: {stats['idle_for']:.2f}s")
```

**Note:** Statistics are for the most recent invocation only. Each new call to the decorated function resets the statistics.

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

## Custom Callbacks

You can write fully custom stop, wait, retry, and lifecycle callbacks.

### Custom Stop Function

A stop function receives `RetryCallState` and returns `True` to stop or `False` to continue.

```python
from tenacity import retry

def stop_on_business_hours(retry_state):
    """Stop retrying outside business hours."""
    from datetime import datetime
    now = datetime.now()
    if now.hour < 8 or now.hour > 18:
        return True  # Stop
    return False  # Continue

@retry(stop=stop_on_business_hours)
def business_operation():
    ...
```

### Custom Wait Function

A wait function receives `RetryCallState` and returns the number of seconds to wait.

```python
from tenacity import retry, stop_after_attempt

def wait_progressive(retry_state):
    """Wait longer for each attempt: 1s, 3s, 7s, 15s, ..."""
    return (2 ** retry_state.attempt_number) - 1

@retry(stop=stop_after_attempt(5), wait=wait_progressive)
def fetch_data():
    ...
```

### Custom Retry Condition

A retry condition receives `RetryCallState` and returns `True` to retry or `False` to stop.

```python
from tenacity import retry, stop_after_attempt

def retry_if_server_error(retry_state):
    """Retry only on 5xx status codes."""
    if retry_state.outcome.failed:
        return True  # Always retry exceptions
    result = retry_state.outcome.result()
    return hasattr(result, 'status_code') and result.status_code >= 500

@retry(stop=stop_after_attempt(3), retry=retry_if_server_error)
def call_api():
    ...
```

### Custom Return Value on Failure

Use `retry_error_callback` to return a custom value when all retries are exhausted, instead of raising `RetryError`.

```python
from tenacity import retry, stop_after_attempt

def return_last_value(retry_state):
    """Return the last result instead of raising RetryError."""
    return retry_state.outcome.result()

def return_default(retry_state):
    """Return a default value on failure."""
    return {"error": "all retries exhausted", "attempts": retry_state.attempt_number}

@retry(
    stop=stop_after_attempt(3),
    retry_error_callback=return_default,
)
def fetch_data():
    ...
```

---

## Complete Code Examples

### HTTP Request Retry with Exponential Backoff

```python
import logging
import requests
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    retry_if_result,
    before_sleep_log,
    RetryError,
)

logger = logging.getLogger(__name__)

def is_server_error(response):
    """Return True if the response is a 5xx server error."""
    return response.status_code >= 500

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=30, jitter=2),
    retry=(
        retry_if_exception_type((requests.ConnectionError, requests.Timeout))
        | retry_if_result(is_server_error)
    ),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
def fetch_with_retry(url: str, **kwargs) -> requests.Response:
    """Fetch a URL with automatic retry on transient failures."""
    response = requests.get(url, timeout=10, **kwargs)
    return response

# Usage:
try:
    resp = fetch_with_retry("https://api.example.com/data")
    data = resp.json()
except requests.ConnectionError:
    print("Service unreachable after 5 attempts")
except requests.Timeout:
    print("Request timed out after 5 attempts")
```

### Database Reconnection

```python
import logging
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log,
)

logger = logging.getLogger(__name__)

# Hypothetical DB exceptions -- replace with your driver's exceptions
class DatabaseConnectionError(Exception):
    pass

class DatabaseTimeoutError(Exception):
    pass

@retry(
    stop=stop_after_attempt(10) | stop_after_delay(300),  # 10 attempts or 5 minutes
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type((DatabaseConnectionError, DatabaseTimeoutError)),
    before_sleep=before_sleep_log(logger, logging.ERROR, exc_info=True),
    reraise=True,
)
def get_db_connection(connection_string: str):
    """Establish a database connection with retry logic."""
    import psycopg2
    try:
        conn = psycopg2.connect(connection_string)
        conn.autocommit = False
        return conn
    except psycopg2.OperationalError as e:
        raise DatabaseConnectionError(str(e)) from e

# Usage:
conn = get_db_connection("postgresql://user:pass@host/db")
```

### Custom Retry Logic Based on Return Value

```python
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_result

def is_not_ready(result: dict) -> bool:
    """Retry if the job status is not 'completed' or 'failed'."""
    return result.get("status") not in ("completed", "failed")

@retry(
    stop=stop_after_attempt(60),
    wait=wait_fixed(5),
    retry=retry_if_result(is_not_ready),
)
def poll_job_status(job_id: str) -> dict:
    """Poll a job until it reaches a terminal state."""
    import requests
    response = requests.get(f"https://api.example.com/jobs/{job_id}")
    return response.json()

# Usage:
result = poll_job_status("job-123")
print(f"Job finished with status: {result['status']}")
```

### Combining Multiple Conditions

```python
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    wait_chain,
    wait_fixed,
    wait_exponential,
    retry_if_exception_type,
    retry_if_exception_message,
    retry_if_result,
)

@retry(
    # Stop: max 8 attempts OR 2 minutes total
    stop=stop_after_attempt(8) | stop_after_delay(120),

    # Wait: fast retries first, then slower
    wait=wait_chain(
        *[wait_fixed(0.5)] * 2,    # First 2 retries: 0.5s
        *[wait_fixed(2)] * 2,      # Next 2 retries: 2s
        wait_exponential(min=5, max=30),  # Remaining: exponential from 5-30s
    ),

    # Retry: on specific transient errors
    retry=(
        retry_if_exception_type((ConnectionError, TimeoutError))
        | retry_if_exception_message(match=r"(rate.limit|throttl|too.many.requests)")
        | retry_if_result(lambda r: r.get("retry") is True)
    ),

    reraise=True,
)
def resilient_operation():
    ...
```

### Logging Retries

```python
import logging
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    before_log,
    after_log,
    before_sleep_log,
)

# Set up logging
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger("retry_demo")

@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    before=before_log(logger, logging.DEBUG),
    after=after_log(logger, logging.DEBUG),
    before_sleep=before_sleep_log(logger, logging.WARNING, exc_info=True),
)
def flaky_operation():
    import random
    if random.random() < 0.7:
        raise ConnectionError("Connection refused")
    return "success"

# Custom callback with detailed logging:
def detailed_log(retry_state):
    if retry_state.outcome is None:
        logger.info(
            "Starting attempt #%d for %s",
            retry_state.attempt_number,
            retry_state.fn.__name__,
        )
        return

    elapsed = retry_state.outcome_timestamp - retry_state.start_time
    if retry_state.outcome.failed:
        logger.warning(
            "Attempt #%d failed after %.2fs: %s",
            retry_state.attempt_number,
            elapsed,
            retry_state.outcome.exception(),
        )
    else:
        logger.info(
            "Attempt #%d succeeded after %.2fs",
            retry_state.attempt_number,
            elapsed,
        )

@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential(min=1, max=10),
    after=detailed_log,
)
def another_flaky_operation():
    ...
```

### Async Retry

```python
import asyncio
import logging
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    before_sleep_log,
)

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=0.5, max=15, jitter=1),
    retry=retry_if_exception_type((ConnectionError, TimeoutError, OSError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def async_fetch_data(url: str) -> dict:
    """Async HTTP fetch with retry."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            if resp.status >= 500:
                raise ConnectionError(f"Server error: {resp.status}")
            resp.raise_for_status()
            return await resp.json()

# Using AsyncRetrying context manager:
async def async_context_manager_example():
    from tenacity import AsyncRetrying, stop_after_attempt, wait_fixed

    async for attempt in AsyncRetrying(
        stop=stop_after_attempt(3),
        wait=wait_fixed(1),
        reraise=True,
    ):
        with attempt:
            result = await some_async_operation()

    return result

# Running:
async def main():
    data = await async_fetch_data("https://api.example.com/data")
    print(data)

asyncio.run(main())
```

---

## Gotchas and Common Mistakes

### 1. Bare `@retry` Retries Forever

```python
# DANGEROUS: retries forever with no wait on ANY exception
@retry
def might_fail():
    ...
```

**Always** specify at least a `stop` strategy in production:

```python
# SAFE: bounded retries
@retry(stop=stop_after_attempt(3))
def might_fail():
    ...
```

Without a `stop`, a permanently failing function creates an infinite loop that consumes CPU.

### 2. Exception Swallowing with `retry_if_result`

When using `retry_if_result`, exceptions are **not retried** unless you also add an exception-based retry condition. This can be surprising.

```python
# BUG: exceptions are NOT retried, only return values are checked
@retry(retry=retry_if_result(lambda r: r is None))
def fetch():
    return requests.get(url)  # If this raises, the exception propagates immediately

# FIX: combine both conditions
@retry(
    retry=(
        retry_if_result(lambda r: r is None)
        | retry_if_exception_type(ConnectionError)
    )
)
def fetch():
    return requests.get(url)
```

### 3. `RetryError` Wraps the Original Exception

By default, when retries are exhausted, tenacity raises `RetryError`, not the original exception. The original exception is accessible via `RetryError.last_attempt.result()` or `.exception()`.

```python
from tenacity import RetryError

try:
    fetch_data()
except RetryError as e:
    original_exception = e.last_attempt.exception()
    print(f"Original error: {original_exception}")
```

Use `reraise=True` to get the original exception directly:

```python
@retry(stop=stop_after_attempt(3), reraise=True)
def fetch_data():
    ...

# Now raises the original exception, not RetryError
```

### 4. Default Behavior When No Stop Is Specified

| Parameter | Default |
|-----------|---------|
| `stop` | `stop_never` (retry forever) |
| `wait` | `wait_none()` (no delay) |
| `retry` | Retry on any exception |
| `before` | `before_nothing` (no-op) |
| `after` | `after_nothing` (no-op) |
| `before_sleep` | `None` (no-op) |
| `reraise` | `False` |

This means `@retry` with no arguments will retry forever, as fast as possible, on any exception. This is almost never the desired behavior.

### 5. Statistics Are Per-Invocation

Statistics are reset on each new call to the decorated function. If you need cumulative statistics, you must track them yourself.

```python
@retry(stop=stop_after_attempt(3))
def fetch():
    ...

fetch()  # statistics has data from this call
fetch()  # statistics is now reset to this call's data only
```

### 6. Interaction with Threading

Tenacity is thread-safe -- each call to a decorated function gets its own `RetryCallState`. However, the `.statistics` attribute on the decorated function is **shared** and will reflect whichever call completed most recently.

```python
# statistics is not thread-safe for reading across concurrent calls
# Each call's retry behavior is independent and thread-safe
```

If you need per-thread statistics, capture them in callbacks:

```python
import threading

thread_stats = threading.local()

def capture_stats(retry_state):
    thread_stats.last_attempts = retry_state.attempt_number
    thread_stats.last_idle = retry_state.idle_for

@retry(stop=stop_after_attempt(3), after=capture_stats)
def fetch():
    ...
```

### 7. Memory Considerations with Retry State

The `RetryCallState` holds references to function arguments and return values. For functions that handle large data:

- The last return value (or exception) is retained until the next call or garbage collection
- For long-running services, consider that `retry_state.outcome` holds a reference to the last result

```python
# If fetch_large_data returns a 1GB object and fails on subsequent
# retries, all intermediate results are released -- only the latest
# outcome is retained in retry_state.
```

### 8. `retry_with()` Creates a New Function Object

The function returned by `retry_with()` is a new object. It does not share statistics or state with the original.

```python
@retry(stop=stop_after_attempt(3))
def fetch():
    ...

fetch_v2 = fetch.retry_with(stop=stop_after_attempt(5))
# fetch and fetch_v2 are completely independent
```

### 9. `wait_chain` -- Last Strategy Repeats

In `wait_chain`, the **last** wait strategy is used for all attempts beyond the defined chain. It does not cycle.

```python
@retry(wait=wait_chain(wait_fixed(1), wait_fixed(2), wait_fixed(5)))
def fetch():
    ...
# Attempt 1 -> wait 1s
# Attempt 2 -> wait 2s
# Attempt 3 -> wait 5s
# Attempt 4 -> wait 5s (last strategy repeats)
# Attempt 5 -> wait 5s
```

### 10. Retry Condition Must Return a Boolean

Custom retry functions must return `True` (retry) or `False` (do not retry). Returning a truthy/falsy non-boolean value works but can be confusing.

```python
# Works but avoid:
@retry(retry=lambda rs: rs.outcome.failed and "timeout" in str(rs.outcome.exception()))

# Better: explicit boolean
def retry_on_timeout(retry_state):
    if not retry_state.outcome.failed:
        return False
    return "timeout" in str(retry_state.outcome.exception())
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

### Callbacks

| Callback | Description |
|----------|-------------|
| `before_log(logger, level)` | Log before each attempt |
| `after_log(logger, level)` | Log after each attempt |
| `before_sleep_log(logger, level, exc_info)` | Log before sleeping between retries |
| Custom `fn(retry_state)` | Any callable accepting `RetryCallState` |
