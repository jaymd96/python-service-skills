# structlog — Examples & Gotchas

> Part of the structlog skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Complete Code Examples](#complete-code-examples)
  - [Development vs. Production Configuration](#development-vs-production-configuration)
  - [Custom Processor: Request Timing](#custom-processor-request-timing)
  - [Web Application Integration (FastAPI/Starlette)](#web-application-integration-fastapistarlette)
- [Gotchas and Common Mistakes](#gotchas-and-common-mistakes)
  - [1. Forgetting merge_contextvars in the Processor Chain](#1-forgetting-merge_contextvars-in-the-processor-chain)
  - [2. Processor Order Matters](#2-processor-order-matters)
  - [3. Mutating the Event Dict in Place](#3-mutating-the-event-dict-in-place)
  - [4. Not Calling configure() Before get_logger()](#4-not-calling-configure-before-get_logger)
  - [5. Mixing Thread-Local and Contextvars](#5-mixing-thread-local-and-contextvars)
  - [6. Serialization Failures in JSONRenderer](#6-serialization-failures-in-jsonrenderer)
  - [7. cache_logger_on_first_use and Testing](#7-cache_logger_on_first_use-and-testing)
  - [8. Log Level Filtering with make_filtering_bound_logger](#8-log-level-filtering-with-make_filtering_bound_logger)

## Complete Code Examples

### Development vs. Production Configuration

```python
import logging
import structlog
import sys

def configure_logging(environment: str = "development"):
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_logger_name,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if environment == "production":
        # JSON output for log aggregation
        structlog.configure(
            processors=shared_processors + [
                structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
            ],
            logger_factory=structlog.stdlib.LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )
        formatter = structlog.stdlib.ProcessorFormatter(
            processors=[
                structlog.stdlib.ProcessorFormatter.remove_processors_meta,
                structlog.processors.JSONRenderer(),
            ],
        )
    else:
        # Colored console output for development
        structlog.configure(
            processors=shared_processors + [
                structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
            ],
            logger_factory=structlog.stdlib.LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )
        formatter = structlog.stdlib.ProcessorFormatter(
            processors=[
                structlog.stdlib.ProcessorFormatter.remove_processors_meta,
                structlog.dev.ConsoleRenderer(),
            ],
        )

    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)
    root = logging.getLogger()
    root.handlers.clear()
    root.addHandler(handler)
    root.setLevel(logging.DEBUG if environment != "production" else logging.INFO)
```

### Custom Processor: Request Timing

```python
import time
import structlog

def add_request_duration(logger, method_name, event_dict):
    """Add request duration if start_time is bound."""
    start = event_dict.pop("_start_time", None)
    if start is not None:
        event_dict["duration_ms"] = round((time.monotonic() - start) * 1000, 2)
    return event_dict

# Usage in a processor chain:
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        add_request_duration,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)

log = structlog.get_logger()
log = log.bind(_start_time=time.monotonic())
# ... do work ...
log.info("request_completed", status=200)
# => {"duration_ms": 42.17, "status": 200, "event": "request_completed", ...}
```

### Web Application Integration (FastAPI/Starlette)

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware

class StructlogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            request_id=request.headers.get("x-request-id", str(uuid.uuid4())),
            method=request.method,
            path=request.url.path,
            client_ip=request.client.host,
        )
        log = structlog.get_logger()
        log.info("request_started")

        response = await call_next(request)

        structlog.contextvars.bind_contextvars(status_code=response.status_code)
        log.info("request_completed")
        return response
```

---

## Gotchas and Common Mistakes

### 1. Forgetting `merge_contextvars` in the Processor Chain

If you use `structlog.contextvars.bind_contextvars()` but do not include `structlog.contextvars.merge_contextvars` in your processor list, the context variables will never appear in log output.

### 2. Processor Order Matters

Processors run in order. If `TimeStamper` is after `JSONRenderer`, the timestamp will not be included. The renderer must always be the last processor in the chain.

```python
# WRONG: renderer is not last
processors=[
    structlog.processors.JSONRenderer(),
    structlog.processors.TimeStamper(fmt="iso"),  # Never reached
]

# CORRECT: renderer is last
processors=[
    structlog.processors.TimeStamper(fmt="iso"),
    structlog.processors.JSONRenderer(),
]
```

### 3. Mutating the Event Dict in Place

Processors receive a mutable dict. Modifying it is the intended pattern, but be careful not to accidentally remove keys that downstream processors expect.

### 4. Not Calling `configure()` Before `get_logger()`

If you call `get_logger()` before `configure()` with `cache_logger_on_first_use=True`, the logger will be cached with default configuration. Later calling `configure()` will not affect cached loggers. Always configure first.

### 5. Mixing Thread-Local and Contextvars

Do not use both `merge_threadlocal` and `merge_contextvars` in the same processor chain. Pick one. For modern Python (3.7+), prefer `contextvars`.

### 6. Serialization Failures in JSONRenderer

If your event dict contains non-serializable objects (datetime, custom classes), `JSONRenderer` will raise a `TypeError`. Add a processor before the renderer to handle serialization:

```python
def serialize_values(logger, method_name, event_dict):
    for key, value in event_dict.items():
        if isinstance(value, datetime):
            event_dict[key] = value.isoformat()
    return event_dict
```

### 7. `cache_logger_on_first_use` and Testing

When testing, set `cache_logger_on_first_use=False` so that configuration changes between tests take effect immediately.

### 8. Log Level Filtering with `make_filtering_bound_logger`

The level passed to `make_filtering_bound_logger()` uses stdlib `logging` level constants (e.g., `logging.INFO`). Passing a string like `"INFO"` will not work.

```python
import logging
# CORRECT:
structlog.make_filtering_bound_logger(logging.INFO)
# WRONG:
structlog.make_filtering_bound_logger("INFO")
```
