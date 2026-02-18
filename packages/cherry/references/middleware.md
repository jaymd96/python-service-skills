# Cherry — Middleware & Observability

> Part of the cherry skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Request Pipeline](#request-pipeline)
- [Built-in Middleware](#built-in-middleware)
- [Structured Logging](#structured-logging)
- [Safety Annotations](#safety-annotations)
- [Enchant Integration](#enchant-integration)

## Request Pipeline

```
Client Request
    |
    v
Request Middleware (runs before routing)
    - Inject loggers into ContextVar
    - Extract IDs (UID, SID, TokenID, OrgID)
    - Create root trace span
    - Set HSTS headers
    |
    v
Routing (match request to handler)
    |
    v
Route Middleware (runs after routing match)
    - Create route span
    - Generate request logs (req2log)
    - Collect metrics
    |
    v
Handler (your application code)
    |
    v
Response (automatic logging and tracing)
```

### Custom Middleware

```python
from py_witchcraft_server.routing.router import Middleware

def my_middleware(request, response, next_handler):
    # Before handler
    start = time.time()
    next_handler()
    # After handler
    duration = time.time() - start
    log.info("request_duration", duration=duration)

# Register on root router
router.add_request_handler_middleware(my_middleware)   # All requests
router.add_route_handler_middleware(my_middleware)     # Matched routes only
```

## Built-in Middleware

10 middleware automatically installed:

| Middleware | Phase | Purpose |
|-----------|-------|---------|
| Logger injection | Request | Inject typed loggers into ContextVar |
| ID extraction | Request | Extract UID, SID, TokenID, OrgID from headers |
| Root trace span | Request | Create Zipkin root span |
| HSTS headers | Request | Set Strict-Transport-Security |
| Route span | Route | Create child trace span for route |
| Request logging | Route | Log via req2log |
| Metrics collection | Route | Record request count, duration |
| Error handling | Route | Catch exceptions, log errors |
| Response logging | Route | Log response status, size |
| Trace completion | Route | Finish trace span |

## Structured Logging

8 Conjure-spec typed loggers, each with versioned schemas:

### svc1log (Service Log)

```python
from py_witchcraft_server.logging import svc1log

logger = svc1log.from_context()
logger.info("Processing request",
    svc1log.safe_param("user_count", 42),
    svc1log.unsafe_param("email", "user@example.com"),
)
logger.warn("High latency", svc1log.safe_param("duration_ms", 500))
logger.error("Database error", svc1log.safe_param("table", "users"))
logger.debug("Query details", svc1log.safe_param("sql", "SELECT ..."))
```

### req2log (Request Log)

```python
from py_witchcraft_server.logging import req2log
# Automatically logged by middleware — includes method, path, status, duration
```

### audit2log / audit3log (Audit Log)

```python
from py_witchcraft_server.logging import audit3log

audit3log.from_context().audit(
    "FILE_ACCESSED",
    audit3log.AuditResult.SUCCESS,
    audit3log.request_param("file_id", "abc123"),
)
# AuditResult: SUCCESS, FAILURE
```

### metric1log (Metric Log)

```python
from py_witchcraft_server.logging import metric1log

metric1log.from_context().metric(
    "api.latency", "timer",
    metric1log.value("duration_ms", 45.2),
)
```

### Other Loggers

| Logger | Purpose |
|--------|---------|
| `trc1log` | Trace logging |
| `evt2log` | Event logging |
| `diag1log` | Diagnostic logging |

### Logger API

All loggers use ContextVar propagation:

```python
logger = svc1log.from_context()  # Get logger with request context
logger.info(message, *params)
logger.warn(message, *params)
logger.error(message, *params)
logger.debug(message, *params)

# Param types
svc1log.safe_param(key, value)     # Logged normally
svc1log.unsafe_param(key, value)   # Redacted in production
```

## Safety Annotations

Safety annotations flow through the entire stack:

```python
# In route registration
router.get("/users/{id}", handler, **safe_path_params("id"))

# In logging
svc1log.safe_param("user_id", 42)       # Always logged
svc1log.unsafe_param("email", "a@b.com") # Redacted in prod -> "<REDACTED>"
```

Production logs redact `unsafe_param` values. Development logs show all values.

## Enchant Integration

Cherry includes an Enchant integration module for generated server stubs:

```python
from py_witchcraft_server.enchant.integration import register_enchant_service

# Register generated service stub with Cherry router
register_enchant_service(
    router=info.router,
    service_impl=CatalogServiceImpl(session_factory),
    # Automatic: error handling, parameter extraction, serialization
)
```

The integration handles:
- Error serialization (Enchant error types -> HTTP responses)
- Parameter extraction (path, query, body -> typed args)
- Response serialization (attrs models -> JSON via cattrs)
- Safety annotation propagation (SafeArg/UnsafeArg -> logging)
