# tenacity — Examples & Gotchas

> Part of the tenacity skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Bare @retry Retries Forever](#1-bare-retry-retries-forever)
  - [2. Exception Swallowing with retry_if_result](#2-exception-swallowing-with-retry_if_result)
  - [3. RetryError Wraps the Original Exception](#3-retryerror-wraps-the-original-exception)
  - [4. Default Behavior When No Stop Is Specified](#4-default-behavior-when-no-stop-is-specified)
  - [5. Statistics Are Per-Invocation](#5-statistics-are-per-invocation)
  - [6. Interaction with Threading](#6-interaction-with-threading)
  - [7. Memory Considerations with Retry State](#7-memory-considerations-with-retry-state)
  - [8. retry_with Creates a New Function Object](#8-retry_with-creates-a-new-function-object)
  - [9. wait_chain -- Last Strategy Repeats](#9-wait_chain----last-strategy-repeats)
  - [10. Retry Condition Must Return a Boolean](#10-retry-condition-must-return-a-boolean)
- [Complete Code Examples](#complete-code-examples)
  - [HTTP Request Retry with Exponential Backoff](#http-request-retry-with-exponential-backoff)
  - [Database Reconnection](#database-reconnection)
  - [Custom Retry Logic Based on Return Value](#custom-retry-logic-based-on-return-value)
  - [Combining Multiple Conditions](#combining-multiple-conditions)
  - [Logging Retries](#logging-retries)
  - [Async Retry](#async-retry)

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
