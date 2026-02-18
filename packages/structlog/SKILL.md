---
name: structlog
description: Structured logging with processor pipelines and context binding. Use when adding structured (JSON) logging, binding request context to loggers, configuring log pipelines, or integrating with stdlib logging. Triggers on structured logging, structlog, JSON logs, log context, processor pipeline, bound logger.
---

# structlog — Structured Logging (v24.4.0)

## Quick Start

```bash
pip install structlog
```

```python
import structlog

log = structlog.get_logger()
log.info("user_logged_in", user_id=42, ip="10.0.0.1")
# {"user_id": 42, "ip": "10.0.0.1", "event": "user_logged_in", "level": "info"}
```

## Key Patterns

### Configuration (call once at startup)
```python
import logging
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer(),  # dev: pretty, prod: JSONRenderer()
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
    cache_logger_on_first_use=True,
)
```

### Context binding and contextvars
```python
log = structlog.get_logger()
log = log.bind(request_id="abc-123", user_id=42)
log.info("processing")  # includes request_id and user_id automatically

# Request-scoped context (propagates across async boundaries)
structlog.contextvars.clear_contextvars()
structlog.contextvars.bind_contextvars(request_id="abc-123")
```

## References

- **[api.md](references/api.md)** — Configuration, processors, stdlib integration, context variables, filtering, async support, and performance tuning
- **[examples.md](references/examples.md)** — Dev/prod configuration, custom processors, FastAPI middleware, and common mistakes
