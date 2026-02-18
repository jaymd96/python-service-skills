# Service-Launcher — Core API Reference

> Part of the service-launcher skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Configuration File](#configuration-file)
- [Launch Modes](#launch-modes)
- [Memory Management](#memory-management)
- [CPU Detection](#cpu-detection)
- [RSS Watchdog](#rss-watchdog)
- [Readiness Probe](#readiness-probe)
- [11-Step Launch Sequence](#11-step-launch-sequence)
- [Platform Support](#platform-support)

## Configuration File

The launcher reads `launcher-static.yml` from the service distribution's `service/run/` directory.

```yaml
# launcher-static.yml
configType: python
launchMode: uvicorn
entryPoint: app:app
args: ["--host", "0.0.0.0", "--port", "8080"]
env:
  LOG_LEVEL: INFO
  PYTHONUNBUFFERED: "1"
memory:
  mode: cgroup-aware
  maxRssPercent: 85
  minReservedMb: 64
watchdog:
  enabled: true
  pollInterval: 10s
  gracePeriod: 30s
readiness:
  enabled: true
  port: 8081
  path: /readiness
  initialDelay: 5s
  timeout: 2s
```

### Top-Level Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `configType` | string | `"python"` | Configuration type (always `python`) |
| `launchMode` | string | `"uvicorn"` | How to start the service (see Launch Modes) |
| `entryPoint` | string | **Required** | Module:callable or script path |
| `args` | list[string] | `[]` | Arguments passed to the service |
| `env` | map[string, string] | `{}` | Environment variables |
| `customConfig` | string | None | Path to custom config to merge |

## Launch Modes

| Mode | `entryPoint` Format | Generated Command |
|------|---------------------|-------------------|
| `pex` | `service.pex` | Runs PEX file directly |
| `module` | `myapp.main` | `python -m myapp.main` |
| `script` | `run.py` | `python run.py` |
| `uvicorn` | `app:app` | `uvicorn app:app --host ... --port ...` |
| `gunicorn` | `app:app` | `gunicorn app:app --bind ...` |
| `command` | `/usr/bin/myapp` | Arbitrary command |

### Mode selection guide

| Use Case | Recommended Mode |
|----------|-----------------|
| ASGI web service (FastAPI, Starlette) | `uvicorn` |
| WSGI web service (Flask, Django) | `gunicorn` |
| PEX-packaged application | `pex` |
| Background worker / CLI tool | `module` or `script` |
| Non-Python binary | `command` |

## Memory Management

```yaml
memory:
  mode: cgroup-aware     # or "static"
  maxRssPercent: 85      # % of cgroup limit
  minReservedMb: 64      # reserved for system
  staticLimitMb: 512     # used when mode=static
```

### Modes

| Mode | Behavior |
|------|----------|
| `cgroup-aware` | Reads cgroup v1/v2 memory limit, computes max RSS as `limit * maxRssPercent / 100 - minReservedMb` |
| `static` | Uses `staticLimitMb` as the hard limit |

### cgroup-aware details

1. Reads `/sys/fs/cgroup/memory/memory.limit_in_bytes` (cgroup v1) or `/sys/fs/cgroup/memory.max` (cgroup v2)
2. Applies `maxRssPercent` (default 85%) to get working limit
3. Subtracts `minReservedMb` (default 64MB) for system overhead
4. Sets `PYTHONMALLOC` and `MALLOC_ARENA_MAX` based on computed limit
5. Passes limit to RSS watchdog

## CPU Detection

The launcher detects available CPUs from:
1. cgroup CPU quota (`cpu.cfs_quota_us / cpu.cfs_period_us`)
2. Falls back to `os.cpu_count()`

Sets `GOMAXPROCS` equivalent and computes default worker counts for uvicorn/gunicorn (`2 * cpu_count + 1`).

## RSS Watchdog

Monitors resident set size and takes action when the process exceeds memory limits.

```yaml
watchdog:
  enabled: true
  pollInterval: 10s     # How often to check RSS
  gracePeriod: 30s      # Time to allow graceful shutdown
  action: restart       # restart | kill | log
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | `true` | Enable RSS monitoring |
| `pollInterval` | duration | `10s` | Check interval |
| `gracePeriod` | duration | `30s` | Graceful shutdown window before SIGKILL |
| `action` | string | `restart` | Action on RSS violation: `restart` (SIGTERM + restart), `kill` (SIGKILL), `log` (warn only) |

### Watchdog flow

```
Poll RSS → Compare to limit → If exceeded:
  → Send SIGTERM → Wait gracePeriod → Send SIGKILL if still running → Restart
```

## Readiness Probe

Lightweight HTTP server for Kubernetes readiness probes.

```yaml
readiness:
  enabled: true
  port: 8081            # Probe port (separate from service port)
  path: /readiness      # Probe path
  initialDelay: 5s      # Wait before first check
  timeout: 2s           # Probe timeout
```

The launcher starts a goroutine serving the readiness endpoint. Returns 200 once the child process is running and responding.

## 11-Step Launch Sequence

```
1. Read config         Read launcher-static.yml + merge customConfig
2. Merge custom        Overlay custom configuration values
3. CPU detection       Detect cgroup CPU quota or os.cpu_count()
4. Memory limits       Compute RSS limit from cgroup or static config
5. Create dirs         Ensure var/logs, var/metrics, var/state exist
6. Set rlimits         Configure NOFILE and other resource limits
7. Build command/env   Construct command line + environment variables
8. Fork                Fork child process with exec
9. PID file            Write child PID to var/state/service.pid
10. Readiness probe    Start readiness HTTP server (if enabled)
11. RSS watchdog       Start RSS monitoring goroutine (if enabled)
```

## Platform Support

Cross-compiled Go binary for 4 platforms:

| OS | Architecture | Binary Name |
|----|-------------|-------------|
| darwin | amd64 | `python-service-launcher` |
| darwin | arm64 | `python-service-launcher` |
| linux | amd64 | `python-service-launcher` |
| linux | arm64 | `python-service-launcher` |

Binaries are distributed via GitHub releases and downloaded by the `[python-service-launcher]` Pants subsystem with SHA256 verification.
