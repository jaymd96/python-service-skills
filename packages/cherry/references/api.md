# Cherry — Core API Reference

> Part of the cherry skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Package Structure](#package-structure)
- [WitchcraftServer](#witchcraftserver)
- [InitContext and RouterInfo](#initcontext-and-routerinfo)
- [Router Protocol](#router-protocol)
- [Health System](#health-system)
- [Configuration](#configuration)
- [Lifecycle](#lifecycle)
- [Metrics](#metrics)
- [Tracing](#tracing)

## Package Structure

Import from `py_witchcraft_server`:

```python
from py_witchcraft_server.server.server import WitchcraftServer
from py_witchcraft_server.server.params import InitContext, RouterInfo, ServerInfo
from py_witchcraft_server.server.lifecycle import ServerState
from py_witchcraft_server.config.install import InstallConfig
from py_witchcraft_server.config.runtime import RuntimeConfig
from py_witchcraft_server.config.refreshable import Refreshable, new_refreshable
from py_witchcraft_server.health.reporter import HealthReporter, new_health_reporter
from py_witchcraft_server.health.status import HealthState, HealthCheckSource
from py_witchcraft_server.metrics.registry import MetricsRegistry, new_metrics_registry
from py_witchcraft_server.tracing.tracer import Tracer, new_tracer
from py_witchcraft_server.tasks.manager import JobManager
```

## WitchcraftServer

Builder pattern for configuring and starting a production-ready HTTPS server.

### Builder Methods

| Method | Description |
|--------|-------------|
| `.with_init_func(fn)` | Set initialization function `(InitContext, RouterInfo) -> Optional[cleanup]` |
| `.with_install_config(config)` | Set InstallConfig directly |
| `.with_install_config_from_file(path)` | Load InstallConfig from YAML |
| `.with_runtime_config(config)` | Set RuntimeConfig directly |
| `.with_runtime_config_refreshable(config)` | Set Refreshable[RuntimeConfig] |
| `.with_runtime_config_from_file(path)` | Load RuntimeConfig from watched YAML (auto-reload) |
| `.with_self_signed_certificate(cn="localhost")` | Generate self-signed cert (dev only) |
| `.with_tls_config(tls)` | Set explicit TLS configuration |
| `.with_health_reporter(reporter)` | Custom HealthReporter |
| `.with_metrics_registry(registry)` | Custom MetricsRegistry |
| `.with_tracer(tracer)` | Custom Tracer |
| `.with_logging()` | Configure structlog JSON backend |

### Lifecycle Methods

| Method | Description |
|--------|-------------|
| `.start()` | Start and block until shutdown |
| `.start_async()` | Start in background, return ServerInfo |
| `.stop(timeout=30.0)` | Graceful shutdown |
| `.wait(timeout=None)` | Wait for server to stop |
| `.state` | Current `ServerState` |
| `.server_info` | `ServerInfo` after start |

## InitContext and RouterInfo

```python
# InitFunc signature
def init_app(ctx: InitContext, info: RouterInfo) -> Callable[[], None] | None:
    # ctx provides:
    ctx.install_config       # InstallConfig
    ctx.runtime_config       # Refreshable[RuntimeConfig]
    ctx.health_reporter      # HealthReporter
    ctx.metrics_registry     # MetricsRegistry
    ctx.job_manager          # JobManager

    # info provides:
    info.router              # Router (register routes here)
    info.context_path        # str (base path prefix)

    return cleanup_fn        # Optional cleanup function
```

### ServerInfo

```python
server_info.address           # str: "0.0.0.0"
server_info.port              # int: 8443
server_info.management_port   # int: 8444
server_info.base_url          # str: "https://0.0.0.0:8443"
server_info.management_url    # str: "https://0.0.0.0:8444"
```

## Router Protocol

```python
from py_witchcraft_server.routing.router import Router, RootRouter

# Route registration
router.register(method, path, handler, **options)
router.get(path, handler, **options)
router.post(path, handler, **options)
router.put(path, handler, **options)
router.delete(path, handler, **options)
router.patch(path, handler, **options)
router.head(path, handler, **options)

# Subrouter with prefix
api = router.subrouter("/api/v1")
api.get("/users", list_users)        # Matches /api/v1/users

# Introspection
router.registered_routes()           # list[RouteSpec]
router.path()                        # str
router.parent()                      # Router | None

# Middleware (RootRouter only)
router.add_request_handler_middleware(*middleware)  # Before routing
router.add_route_handler_middleware(*middleware)    # After routing match
```

### Route Options

```python
from py_witchcraft_server.routing.router import (
    safe_path_params, safe_query_params, safe_header_params,
    forbidden_path_params, forbidden_query_params,
    disable_telemetry, metric_tags,
)

router.get("/users/{id}", handler, **safe_path_params("id"))
router.get("/search", handler, **safe_query_params("q", "page"), **metric_tags(endpoint="search"))
```

## Health System

### HealthState Enum

| State | HTTP Code | Description |
|-------|-----------|-------------|
| `HEALTHY` | 200 | Fully operational |
| `DEFERRING` | 518 | Operational, requesting to defer shutdown |
| `SUSPENDED` | 519 | No longer serving, ready for shutdown |
| `REPAIRING` | 520 | Degraded, capable of auto-recovery |
| `WARNING` | 521 | Trending towards error |
| `ERROR` | 522 | Operationally unhealthy |
| `TERMINAL` | 523 | Unrecoverable |

### HealthReporter

```python
reporter = new_health_reporter()
component = reporter.initialize_health_component("DATABASE")

component.healthy()
component.error(Exception("Connection timeout"))
component.warning("High latency detected")
component.repairing("Reconnecting...")
```

### Built-in Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /status/liveness` | Basic liveness probe |
| `GET /status/readiness` | Traffic readiness probe |
| `GET /status/health` | Comprehensive health checks (optional auth) |
| `GET /debug/diagnostic/{type}` | Runtime diagnostics |

## Configuration

### InstallConfig (var/conf/install.yml)

```yaml
product-name: my-service
product-version: 1.0.0
server:
  port: 8443
  management-port: 8444
  address: "0.0.0.0"
  context-path: "/"
  cert-file: var/conf/cert.pem
  key-file: var/conf/key.pem
metrics-emit-frequency: 60s
trace-sample-rate: 1.0
```

### RuntimeConfig (var/conf/runtime.yml)

```yaml
logging:
  level: info
health-checks:
  shared-secret: changeme
diagnostics:
  debug-shared-secret: changeme
```

### Refreshable[T]

Runtime config supports auto-reload from file:

```python
server.with_runtime_config_from_file("var/conf/runtime.yml")
# File changes are detected and config reloaded automatically
```

## Lifecycle

### ServerState Enum

`UNINITIALIZED` -> `INITIALIZING` -> `RUNNING` -> `DRAINING` -> `STOPPED`

Signal handlers (SIGTERM, SIGINT) trigger graceful shutdown with configurable drain timeout.

## Metrics

```python
registry = new_metrics_registry()
counter = registry.counter("requests.total")
timer = registry.timer("request.duration")
gauge = registry.gauge("connections.active")
histogram = registry.histogram("response.size")
meter = registry.meter("events.per_second")
```

5 metric types: counter, timer, gauge, histogram, meter.

## Tracing

```python
from py_witchcraft_server.tracing.tracer import new_tracer, trace_sampler_rate

tracer = new_tracer(trace_sampler_rate(1.0))
span = tracer.start_span_from_context("database_query")
span.tag("rows", 100)
span.finish()
```

B3/Zipkin header propagation: `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled`.
