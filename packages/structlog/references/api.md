# structlog — API Reference

> Part of the structlog skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents

- [Core API](#core-api)
  - [structlog.configure()](#structlogconfigure)
  - [structlog.get_logger()](#structlogget_logger)
  - [BoundLogger and Context Binding](#boundlogger-and-context-binding)
- [Processor Pipeline Architecture](#processor-pipeline-architecture)
  - [Dropping Events](#dropping-events)
- [Built-in Processors](#built-in-processors)
  - [structlog.processors.add_log_level](#structlogprocessorsadd_log_level)
  - [structlog.processors.TimeStamper](#structlogprocessorstimestamper)
  - [structlog.processors.JSONRenderer](#structlogprocessorsjsonrenderer)
  - [structlog.dev.ConsoleRenderer](#structlogdevconsolerenderer)
  - [structlog.processors.StackInfoRenderer](#structlogprocessorsstackinforenderer)
  - [structlog.processors.format_exc_info](#structlogprocessorsformat_exc_info)
  - [structlog.processors.UnicodeDecoder](#structlogprocessorsunicodedecoder)
  - [structlog.processors.ExceptionPrettyPrinter](#structlogprocessorsexceptionprettyprinter)
- [stdlib Integration](#stdlib-integration)
  - [Using structlog with stdlib Logging](#using-structlog-with-stdlib-logging)
  - [ProcessorFormatter for stdlib Handlers](#processorformatter-for-stdlib-handlers)
  - [Rendering stdlib Logs as Structured Output](#rendering-stdlib-logs-as-structured-output)
- [Context Variables (structlog.contextvars)](#context-variables-structlogcontextvars)
  - [Middleware Pattern (e.g., ASGI/WSGI)](#middleware-pattern-eg-asgiwsgi)
- [Filtering by Log Level](#filtering-by-log-level)
  - [Using make_filtering_bound_logger](#using-make_filtering_bound_logger)
  - [Using a Filter Processor](#using-a-filter-processor)
- [Thread-Local Context](#thread-local-context)
- [Async Support](#async-support)
- [Performance](#performance)

## Core API

### `structlog.configure()`

The global configuration function. Call it once at application startup to define how all structlog loggers behave.

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.dev.set_exc_info,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)
```

#### Key Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `processors` | `list[Processor]` | Ordered list of processor callables applied to every log event |
| `wrapper_class` | `type` | The bound logger class returned by `get_logger()` |
| `context_class` | `type` | Dictionary type for the event dict (usually `dict`) |
| `logger_factory` | `callable` | Factory that creates the underlying logger instance |
| `cache_logger_on_first_use` | `bool` | Cache the configured logger after first use for performance |

### `structlog.get_logger()`

The primary entry point for obtaining a logger.

```python
import structlog

log = structlog.get_logger()
log.info("user_logged_in", user_id=42, ip="10.0.0.1")
# => {"user_id": 42, "ip": "10.0.0.1", "event": "user_logged_in", "level": "info"}
```

You can pass initial bindings:

```python
log = structlog.get_logger(service="auth", version="2.1")
log.info("started")
# => {"service": "auth", "version": "2.1", "event": "started", "level": "info"}
```

### `BoundLogger` and Context Binding

Bound loggers carry context that is included in every subsequent log call. The `bind()` method returns a new logger with additional context, while `unbind()` removes keys.

```python
log = structlog.get_logger()
log = log.bind(request_id="abc-123")
log.info("processing")
# => {"request_id": "abc-123", "event": "processing", "level": "info"}

log = log.bind(user_id=42)
log.info("authorized")
# => {"request_id": "abc-123", "user_id": 42, "event": "authorized", "level": "info"}

log = log.unbind("user_id")
log.info("done")
# => {"request_id": "abc-123", "event": "done", "level": "info"}
```

**`new()`** creates a new logger with only the provided bindings (clears all existing context):

```python
log = log.new(request_id="def-456")
log.info("fresh_context")
# => {"request_id": "def-456", "event": "fresh_context", "level": "info"}
```

---

## Processor Pipeline Architecture

Processors are the heart of structlog. Each processor is a callable with the signature:

```python
def my_processor(logger, method_name, event_dict):
    """
    Args:
        logger: The wrapped logger object.
        method_name: The name of the method called (e.g., "info", "error").
        event_dict: The mutable dictionary of the log event.

    Returns:
        The (possibly modified) event_dict, or raises DropEvent to discard.
    """
    event_dict["custom_key"] = "custom_value"
    return event_dict
```

Processors execute in order. The output of one becomes the input of the next. The final processor typically renders the event dict into a string or bytes.

```
bind(key=val) -> event_dict
    |
    v
[processor_1] -> [processor_2] -> ... -> [renderer] -> output
```

### Dropping Events

Raise `structlog.DropEvent` from any processor to silently discard a log event:

```python
import structlog

def drop_health_checks(logger, method_name, event_dict):
    if event_dict.get("event") == "health_check":
        raise structlog.DropEvent
    return event_dict
```

---

## Built-in Processors

### `structlog.processors.add_log_level`

Adds the `level` key to the event dict based on the method name called.

```python
structlog.processors.add_log_level
# log.info("x") => {"event": "x", "level": "info"}
# log.error("y") => {"event": "y", "level": "error"}
```

### `structlog.processors.TimeStamper`

Adds a timestamp to the event dict.

```python
structlog.processors.TimeStamper(fmt="iso")        # ISO 8601 string
structlog.processors.TimeStamper(fmt="iso", utc=True)  # UTC ISO 8601
structlog.processors.TimeStamper(fmt="%Y-%m-%d")    # Custom strftime format
structlog.processors.TimeStamper()                  # UNIX timestamp (float)
```

The timestamp is added under the `"timestamp"` key by default. Use the `key` parameter to change it:

```python
structlog.processors.TimeStamper(fmt="iso", key="ts")
```

### `structlog.processors.JSONRenderer`

Serializes the event dict to a JSON string. This is typically the final processor.

```python
structlog.processors.JSONRenderer(indent=None, sort_keys=False)
```

```python
log.info("request", method="GET", path="/api")
# => {"event": "request", "method": "GET", "path": "/api", "level": "info"}
```

### `structlog.dev.ConsoleRenderer`

Renders human-readable, optionally colored output for development. Not suitable for production log aggregation.

```python
structlog.dev.ConsoleRenderer(
    colors=True,           # Enable ANSI colors
    pad_event=30,          # Pad event name for alignment
    repr_native_str=False,
)
```

Output example:
```
2024-01-15 10:23:45 [info     ] request_started   method=GET path=/api
```

### `structlog.processors.StackInfoRenderer`

Renders the `stack_info` key (if present) into a formatted stack trace string.

```python
structlog.processors.StackInfoRenderer()
```

### `structlog.processors.format_exc_info`

Formats exception info from the `exc_info` key into a human-readable traceback string.

```python
structlog.processors.format_exc_info
```

Usage:

```python
try:
    1 / 0
except ZeroDivisionError:
    log.error("calculation_failed", exc_info=True)
```

### `structlog.processors.UnicodeDecoder`

Decodes byte strings in the event dict to Unicode:

```python
structlog.processors.UnicodeDecoder()
```

### `structlog.processors.ExceptionPrettyPrinter`

Pretty-prints exceptions. Useful as a processor before `ConsoleRenderer`.

---

## stdlib Integration

structlog can operate as a complete replacement for stdlib logging or as a layer on top of it.

### Using structlog with stdlib Logging

The most common production pattern routes structlog output through stdlib logging handlers:

```python
import logging
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.stdlib.add_logger_name,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
    ],
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)
```

### `ProcessorFormatter` for stdlib Handlers

`ProcessorFormatter` is a `logging.Formatter` subclass that lets you use structlog processors with any stdlib handler. This is how you get JSON output from `FileHandler`, `StreamHandler`, etc.

```python
import logging
import structlog

formatter = structlog.stdlib.ProcessorFormatter(
    processors=[
        structlog.stdlib.ProcessorFormatter.remove_processors_meta,
        structlog.processors.JSONRenderer(),
    ],
)

handler = logging.StreamHandler()
handler.setFormatter(formatter)

root_logger = logging.getLogger()
root_logger.addHandler(handler)
root_logger.setLevel(logging.INFO)
```

This lets stdlib log records also pass through structlog processors, unifying the output format of all logging in your application (including third-party libraries that use stdlib logging).

### Rendering stdlib Logs as Structured Output

```python
# Foreign (non-structlog) log entries also get formatted:
logging.getLogger("urllib3").info("Connection established")
# => {"event": "Connection established", "logger": "urllib3", "level": "info"}
```

---

## Context Variables (`structlog.contextvars`)

Context variables allow you to bind context that automatically propagates across async boundaries and is available to all loggers within the same context (e.g., a web request).

```python
import structlog

# Add to your processor chain:
# structlog.contextvars.merge_contextvars

# Bind context for the current execution context:
structlog.contextvars.clear_contextvars()
structlog.contextvars.bind_contextvars(request_id="abc-123", user_id=42)

log = structlog.get_logger()
log.info("handling_request")
# => {"request_id": "abc-123", "user_id": 42, "event": "handling_request"}

# Clear when done:
structlog.contextvars.unbind_contextvars("request_id", "user_id")
```

### Middleware Pattern (e.g., ASGI/WSGI)

```python
import structlog

async def logging_middleware(request, call_next):
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request.headers.get("x-request-id", "unknown"),
        method=request.method,
        path=request.url.path,
    )
    response = await call_next(request)
    structlog.contextvars.bind_contextvars(status=response.status_code)
    return response
```

All loggers used within that request context (even in deeply nested functions) will automatically include the bound context variables.

---

## Filtering by Log Level

### Using `make_filtering_bound_logger`

The most performant way to filter by log level. Events below the threshold are discarded before any processors run.

```python
import logging
import structlog

structlog.configure(
    wrapper_class=structlog.make_filtering_bound_logger(logging.WARNING),
    # ...
)

log = structlog.get_logger()
log.debug("ignored")    # Dropped immediately, zero processing cost
log.warning("kept")     # Processed normally
```

### Using a Filter Processor

For dynamic or conditional filtering:

```python
def filter_by_level(logger, method_name, event_dict):
    import logging
    level = event_dict.get("level", "debug")
    if getattr(logging, level.upper()) < logging.WARNING:
        raise structlog.DropEvent
    return event_dict
```

---

## Thread-Local Context

For applications not using asyncio, thread-local context provides per-thread bindings:

```python
import structlog

structlog.configure(
    processors=[
        structlog.threadlocal.merge_threadlocal,
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
    # ...
)

structlog.threadlocal.clear_threadlocal()
structlog.threadlocal.bind_threadlocal(request_id="xyz")

log = structlog.get_logger()
log.info("threaded")
# => {"request_id": "xyz", "event": "threaded", "level": "info"}
```

**Note:** `contextvars` (introduced in Python 3.7) is preferred over thread-local in most cases, as it works with both threads and asyncio.

---

## Async Support

structlog works natively with async code. The `contextvars` integration propagates context through `await` chains and `asyncio.Task` boundaries.

```python
import asyncio
import structlog

log = structlog.get_logger()

async def handle_request(request_id: str):
    structlog.contextvars.bind_contextvars(request_id=request_id)
    log.info("started")
    await process_data()
    log.info("completed")

async def process_data():
    # request_id is automatically available via contextvars
    log = structlog.get_logger()
    log.info("processing")
```

---

## Performance

structlog is designed for high-throughput logging. Key performance features:

- **`cache_logger_on_first_use=True`**: Caches the fully configured logger after the first log call, avoiding repeated processor chain construction.
- **`make_filtering_bound_logger()`**: Drops below-threshold events at the C level before any Python processor code runs.
- **Lazy evaluation**: Processors only run when a log event passes filtering.
- **`structlog.WriteLoggerFactory`**: Writes directly to a file-like object, bypassing stdlib logging overhead entirely.
