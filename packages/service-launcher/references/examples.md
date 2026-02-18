# Service-Launcher — Examples & Gotchas

> Part of the service-launcher skill. See [SKILL.md](../SKILL.md) for overview.

## Uvicorn ASGI Service

```yaml
# launcher-static.yml
configType: python
launchMode: uvicorn
entryPoint: app:app
args: ["--host", "0.0.0.0", "--port", "8080", "--workers", "4"]
memory:
  mode: cgroup-aware
  maxRssPercent: 85
watchdog:
  enabled: true
  pollInterval: 10s
readiness:
  enabled: true
  port: 8081
```

## Gunicorn WSGI Service

```yaml
configType: python
launchMode: gunicorn
entryPoint: myapp.wsgi:application
args: ["--bind", "0.0.0.0:8080", "--workers", "4", "--timeout", "120"]
memory:
  mode: cgroup-aware
  maxRssPercent: 80
watchdog:
  enabled: true
  pollInterval: 15s
  gracePeriod: 60s
```

## PEX Application

```yaml
configType: python
launchMode: pex
entryPoint: my-service.pex
args: ["--config", "/opt/services/config.yml"]
memory:
  mode: static
  staticLimitMb: 256
watchdog:
  enabled: false
```

## Background Worker

```yaml
configType: python
launchMode: module
entryPoint: myworker.main
args: ["--queue", "default", "--concurrency", "4"]
env:
  REDIS_URL: "redis://redis:6379"
  LOG_LEVEL: "INFO"
memory:
  mode: cgroup-aware
  maxRssPercent: 90
  minReservedMb: 32
watchdog:
  enabled: true
  pollInterval: 30s
  action: log
```

## Memory-Constrained Environment

```yaml
configType: python
launchMode: uvicorn
entryPoint: api:app
memory:
  mode: cgroup-aware
  maxRssPercent: 75        # Conservative — leave room for spikes
  minReservedMb: 128       # More reserved for sidecar containers
watchdog:
  enabled: true
  pollInterval: 5s         # Check more frequently
  gracePeriod: 15s         # Shorter grace — fail fast
  action: restart
```

## Production Service with All Options

```yaml
configType: python
launchMode: uvicorn
entryPoint: catalog.app:app
args:
  - "--host"
  - "0.0.0.0"
  - "--port"
  - "8080"
  - "--workers"
  - "4"
  - "--log-level"
  - "info"
env:
  PYTHONUNBUFFERED: "1"
  DB_HOST: "postgres.internal"
  DB_POOL_SIZE: "10"
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
memory:
  mode: cgroup-aware
  maxRssPercent: 85
  minReservedMb: 64
watchdog:
  enabled: true
  pollInterval: 10s
  gracePeriod: 30s
  action: restart
readiness:
  enabled: true
  port: 8081
  path: /readiness
  initialDelay: 10s
  timeout: 3s
```

## Corresponding BUILD Target

```python
# BUILD
sls_service(
    name="catalog-service",
    product_group="com.example",
    product_name="catalog-service",
    version="1.0.0",
    entrypoint="catalog.app:app",
    command="uvicorn",
    args=["--host", "0.0.0.0", "--port", "8080", "--workers", "4"],
    check_args=["--check"],
    resource_requests={"cpu": "100m", "memory": "256Mi"},
    resource_limits={"cpu": "500m", "memory": "512Mi"},
)
```

The `sls-package` goal generates `launcher-static.yml` from these BUILD fields automatically.

## Gotchas

1. **Go binary, not Python**: The service launcher is a Go binary that _launches_ Python services. You don't import it — it's bundled into the SLS distribution and invoked as the container entrypoint
2. **cgroup-aware mode requires containers**: `cgroup-aware` memory mode reads cgroup limits from `/sys/fs/cgroup/`. Outside containers (local dev), use `static` mode with `staticLimitMb`
3. **Readiness port must differ from service port**: The readiness probe runs on a separate port (e.g., 8081) — not on the main service port (8080)
4. **Watchdog `restart` re-forks**: On RSS violation with `action: restart`, the launcher sends SIGTERM, waits `gracePeriod`, sends SIGKILL if needed, then forks a new child. The launcher process itself stays running
5. **Worker count auto-tuning**: If `args` doesn't specify `--workers`, uvicorn/gunicorn modes auto-compute workers from detected CPU count (`2 * cpus + 1`). Set explicitly to override
6. **`launcher-static.yml` is generated**: Don't hand-write it. The `sls-package` goal generates it from BUILD target fields. Use `customConfig` to merge additional settings
7. **Platform binaries are SHA256-verified**: The `[python-service-launcher]` subsystem downloads binaries with checksum verification. Invalid hashes cause build failure
