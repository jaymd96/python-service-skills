---
name: tenacity
description: Composable retry logic for Python. Use when adding retry behavior, exponential backoff, configuring stop/wait/retry conditions, or building resilient service calls. Triggers on retry, tenacity, backoff, retry logic, resilience, stop_after_attempt, wait_exponential.
---

# tenacity — Retry Logic (v9.1.2)

## Quick Start

```bash
pip install tenacity
```

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def fetch_data():
    response = httpx.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()
```

## Key Patterns

### Composable primitives (combine with `|` and `+`)
```python
@retry(
    stop=stop_after_attempt(5) | stop_after_delay(30),    # OR
    wait=wait_exponential(multiplier=1, max=60),
    retry=retry_if_exception_type((ConnectionError, TimeoutError)),
)
```

### Retry on return value
```python
from tenacity import retry_if_result

@retry(retry=retry_if_result(lambda r: r is None), stop=stop_after_attempt(5))
def poll_status():
    return get_status()  # retries while None
```

## References

- **[api.md](references/api.md)** — Core retry API, wait/stop/retry strategies, combining conditions
- **[callbacks.md](references/callbacks.md)** — Before/after callbacks, logging, retry statistics, async support
- **[examples.md](references/examples.md)** — Complete examples, gotchas, common patterns

## Grep Patterns

- `@retry` — Find all retry-decorated functions
- `wait_exponential|wait_fixed` — Find wait strategy usage
- `stop_after_attempt` — Find stop conditions
- `retry_if_exception_type` — Find retry conditions
