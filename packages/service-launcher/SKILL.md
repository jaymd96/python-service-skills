---
name: service-launcher
description: Go binary for running Python services in containers. Use when configuring launch modes (pex, uvicorn, gunicorn), memory management (cgroup-aware RSS watchdog), CPU detection, readiness probes, or launcher YAML configuration. Cross-compiled for darwin/linux x amd64/arm64. Triggers on service-launcher, launcher, RSS watchdog, cgroup memory, launch mode, readiness probe, launcher-static.yml.
---

# service-launcher — Container Process Manager (v0.1.0)

## Quick Start

The service-launcher is a Go binary bundled into SLS distributions. It reads `launcher-static.yml` and launches the Python service with proper resource management.

```yaml
# launcher-static.yml
configType: python
launchMode: uvicorn
entryPoint: app:app
args: ["--host", "0.0.0.0", "--port", "8080"]
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

## Key Patterns

### 11-step launch sequence
```
Read config -> Merge custom -> CPU detection -> Memory limits -> Create dirs
-> Set rlimits -> Build command/env -> Fork -> PID file -> Readiness probe -> RSS watchdog
```

### 6 launch modes
| Mode | Command |
|------|---------|
| `pex` | Runs PEX file directly |
| `module` | `python -m module_name` |
| `script` | `python script.py` |
| `uvicorn` | `uvicorn entrypoint --host ... --port ...` |
| `gunicorn` | `gunicorn entrypoint --bind ...` |
| `command` | Arbitrary command |

## References

- **[api.md](references/api.md)** — Configuration YAML, launch modes, memory/CPU management, watchdog, readiness
- **[examples.md](references/examples.md)** — Configuration examples, memory tuning, launch mode selection, gotchas
