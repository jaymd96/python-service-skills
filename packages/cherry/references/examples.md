# Cherry — Examples & Gotchas

> Part of the cherry skill. See [SKILL.md](../SKILL.md) for overview.

## Minimal Server with Health Check

```python
from py_witchcraft_server.server.server import WitchcraftServer
from py_witchcraft_server.server.params import InitContext, RouterInfo

def init_app(ctx: InitContext, info: RouterInfo):
    db_health = ctx.health_reporter.initialize_health_component("DATABASE")
    db_health.healthy()

    @info.router.get("/api/greeting")
    def greeting():
        return {"message": "Hello, World!"}

    return None

server = (
    WitchcraftServer()
    .with_logging()
    .with_init_func(init_app)
    .with_self_signed_certificate()
)
server.start()
```

## Server with Config Files

```python
server = (
    WitchcraftServer()
    .with_logging()
    .with_init_func(init_app)
    .with_install_config_from_file("var/conf/install.yml")
    .with_runtime_config_from_file("var/conf/runtime.yml")
    .with_tls_config(TLSConfig(
        cert_file="/etc/ssl/cert.pem",
        key_file="/etc/ssl/key.pem",
    ))
)
```

## Subrouter for API Versioning

```python
def init_app(ctx: InitContext, info: RouterInfo):
    v1 = info.router.subrouter("/api/v1")
    v1.get("/users", list_users_v1)
    v1.post("/users", create_user_v1)

    v2 = info.router.subrouter("/api/v2")
    v2.get("/users", list_users_v2)

    return None
```

## Health Check Implementation

```python
def init_app(ctx: InitContext, info: RouterInfo):
    db_health = ctx.health_reporter.initialize_health_component("DATABASE")
    cache_health = ctx.health_reporter.initialize_health_component("CACHE")

    # Background health monitoring
    def check_health():
        try:
            db.execute("SELECT 1")
            db_health.healthy()
        except Exception as e:
            db_health.error(e)

        try:
            cache.ping()
            cache_health.healthy()
        except Exception:
            cache_health.warning("Cache unavailable, using fallback")

    ctx.job_manager.schedule(check_health, interval_seconds=30)
    return None
```

## Background Job with JobManager

```python
def init_app(ctx: InitContext, info: RouterInfo):
    def emit_metrics():
        count = get_active_connections()
        ctx.metrics_registry.gauge("connections.active").set(count)

    ctx.job_manager.schedule(emit_metrics, interval_seconds=60)

    def cleanup():
        ctx.job_manager.stop()

    return cleanup
```

## Testing with start_async

```python
def test_server():
    server = (
        WitchcraftServer()
        .with_init_func(init_app)
        .with_self_signed_certificate()
    )
    info = server.start_async()

    try:
        import httpx
        r = httpx.get(f"{info.base_url}/api/greeting", verify=False)
        assert r.status_code == 200
        assert r.json()["message"] == "Hello, World!"
    finally:
        server.stop(timeout=5.0)
```

## Gotchas

1. **TLS is expected**: Production requires TLS. Use `.with_self_signed_certificate()` for development only
2. **HTTPS enforced**: Witchcraft is opinionated — plain HTTP is a warning, not the default
3. **Use info.router, not CherryPy directly**: Route through the Router protocol for middleware, logging, and tracing to work
4. **ContextVar propagation**: Loggers use `from_context()` — they must be called within a request context (middleware sets this up)
5. **Management port**: Can be separate from app port for health checks behind a load balancer
6. **Cleanup function order**: Cleanup runs LIFO — app cleanup first, then job manager, then CherryPy
7. **Observability is mandatory**: Logging, tracing, and metrics are built in and cannot be disabled
