# tenacity — Callbacks & Advanced

> Part of the tenacity skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Before/After Callbacks](#beforeafter-callbacks)
  - [before_log](#before_loglogger-log_level)
  - [after_log](#after_loglogger-log_level)
  - [before_sleep_log](#before_sleep_loglogger-log_level-exc_infofalse)
  - [Custom Callbacks](#custom-callbacks)
- [RetryCallState](#retrycallstate)
  - [Key Attributes](#key-attributes)
  - [Accessing Outcome Data](#accessing-outcome-data)
- [Custom Callbacks (Advanced)](#custom-callbacks-advanced)
  - [Custom Stop Function](#custom-stop-function)
  - [Custom Wait Function](#custom-wait-function)
  - [Custom Retry Condition](#custom-retry-condition)
  - [Custom Return Value on Failure](#custom-return-value-on-failure)
- [Statistics](#statistics)
- [Async Support](#async-support)
  - [Async Context Manager](#async-context-manager)
  - [Async Event-Based Stop](#async-event-based-stop)
  - [Async Sleep Override](#async-sleep-override)

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

### Custom Callbacks (Advanced)

You can write fully custom stop, wait, retry, and lifecycle callbacks.

#### Custom Stop Function

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

#### Custom Wait Function

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

#### Custom Retry Condition

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

#### Custom Return Value on Failure

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
