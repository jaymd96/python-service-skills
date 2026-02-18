---
name: cherry
description: Opinionated Python server framework (Witchcraft port). Use when building production services with WitchcraftServer builder, Router protocol, structured Conjure-spec logging (svc1log, req2log, audit3log), health checks (7 HealthStates), B3/Zipkin tracing, metrics, middleware chains, TLS, or Enchant integration. Triggers on cherry, witchcraft, WitchcraftServer, svc1log, health reporter, Router, middleware, CherryPy.
---

# Cherry — Witchcraft Server Framework (v0.1.0)

## Quick Start

```bash
pip install py-witchcraft-server
```

```python
from py_witchcraft_server.server.server import WitchcraftServer
from py_witchcraft_server.server.params import InitContext, RouterInfo

def init_app(ctx: InitContext, info: RouterInfo):
    @info.router.get("/hello")
    def hello():
        return {"message": "Hello, World!"}
    return None  # optional cleanup function

server = (
    WitchcraftServer()
    .with_init_func(init_app)
    .with_self_signed_certificate()
)
server.start()
```

## Key Patterns

### Health checks
```python
from py_witchcraft_server.health.reporter import new_health_reporter
from py_witchcraft_server.health.status import HealthState

reporter = new_health_reporter()
db_health = reporter.initialize_health_component("DATABASE")
db_health.healthy()
db_health.error(Exception("Connection timeout"))
# 7 states: HEALTHY, DEFERRING, SUSPENDED, REPAIRING, WARNING, ERROR, TERMINAL
```

### Structured logging (Conjure-spec)
```python
from py_witchcraft_server.logging import svc1log, req2log, audit3log, metric1log

svc1log.from_context().info("msg", svc1log.safe_param("k", v), svc1log.unsafe_param("email", e))
audit3log.from_context().audit("FILE_ACCESSED", audit3log.AuditResult.SUCCESS)
```

## References

- **[api.md](references/api.md)** — WitchcraftServer builder, Router protocol, health, config, lifecycle
- **[middleware.md](references/middleware.md)** — Logging, tracing, metrics, middleware chains, Enchant integration
- **[examples.md](references/examples.md)** — Complete server setup, health checks, testing patterns, gotchas

## Grep Patterns

- `WitchcraftServer|with_init_func` — Find server setup
- `svc1log|req2log|audit3log` — Find structured logging
- `health_reporter|HealthState` — Find health check usage
- `\.get\(|\.post\(|\.register\(` — Find route registration
