# Forge Pipeline — Pipeline Details

> Part of the forge-pipeline workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Pipeline Architecture](#pipeline-architecture)
- [Stage Details](#stage-details)
- [Tool Interactions](#tool-interactions)
- [Generated Distribution Layout](#generated-distribution-layout)
- [Hook Init System](#hook-init-system)
- [Docker Generation](#docker-generation)
- [Publishing Flow](#publishing-flow)
- [Service Launcher Integration](#service-launcher-integration)

## Pipeline Architecture

```
                    sls-distribution (Pants plugin)
                    ┌─────────────────────────────────┐
  BUILD target      │                                 │
  sls_service() ──→ │  sls-manifest → sls-validate   │
                    │       ↓                         │
                    │  sls-package ──→ sls-lock        │
                    │       ↓                         │
                    │  sls-docker ──→ sls-publish      │
                    └───┬────────────────┬────────────┘
                        │                │
            ┌───────────┘                └──────────┐
            ↓                                       ↓
    docker-generator                          release-hub
    (Dockerfile + .dockerignore)              (Apollo Hub client)
            ↓
    service-launcher
    (Go binary bundled into distribution)
```

## Stage Details

### Stage 1-2: Setup & Define

Configure `pants.toml` and create `sls_service` BUILD targets. See SKILL.md Quick Start.

### Stage 3: Manifest Generation (`sls-manifest`)

Reads all fields from the `sls_service` target and generates `deployment/manifest.yml`:

```yaml
manifest-version: "1.0"
product-type: helm.v1
product-group: com.example
product-name: catalog-service
product-version: 1.0.0
display-name: Catalog Service
traits: [api, web]
labels:
  team: platform
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { cpu: 500m, memory: 512Mi }
replication:
  desired: 2
  min: 1
  max: 5
extensions:
  product-dependencies: [...]
  artifacts: [...]
```

### Stage 4: Validation (`sls-validate`)

Two validation layers:
1. **Schema validation** — JSON schema (controlled by `strict_validation`)
2. **Semantic validation**:
   - Product identity format (`^[a-z0-9.-]+$`, `^[a-z][a-z0-9.-]*$`)
   - Version format (SLS orderable only)
   - Replication constraints (`min <= desired <= max`)
   - Dependency constraints (no lockstep, no duplicates)
   - Health check mutual exclusivity

### Stage 5: Package (`sls-package`)

Creates the full distribution layout and packages as `.sls.tgz`:

1. Creates directory structure (see layout below)
2. Downloads service-launcher binaries (4 platforms, SHA256-verified)
3. Generates PEX binary from Python sources
4. Generates `launcher-static.yml` from BUILD fields
5. Generates init script
6. Generates health check scripts (if configured)
7. Copies hook scripts (if configured)
8. Generates lock file (if dependencies exist)
9. Creates tarball

### Stage 6: Docker (`sls-docker`)

Calls `docker-generator.sls_dockerfile()` to generate:
- `docker/Dockerfile` — Multi-platform, SLS-aware
- `docker/.dockerignore` — SLS-specific ignores
- `docker/hooks/entrypoint.sh` — Hook lifecycle entrypoint (if hooks enabled)
- `docker/hooks/hooks.sh` — Hook execution library (if hooks enabled)

### Stage 7: Publish (`sls-publish`)

Calls `release-hub.ApolloHubClient.publish_release()`:
1. Auto-detects version format → release type → channel
2. Reads manifest YAML
3. Constructs `PublishRequest` with all metadata
4. POSTs to Apollo Hub `/api/v1/releases`
5. Reports success/failure

## Tool Interactions

### sls-package → service-launcher

The `[python-service-launcher]` subsystem downloads Go binaries:

```toml
[python-service-launcher]
version = "v0.1.0"
github_repo = "jaymd96/python-service-launcher"
```

Downloads 4 platform variants (`darwin-amd64`, `darwin-arm64`, `linux-amd64`, `linux-arm64`), verifies SHA256 checksums, and places them in `service/bin/<os>-<arch>/python-service-launcher`.

### sls-docker → docker-generator

The `sls-docker` goal calls `sls_dockerfile()` from docker-generator:

```python
from pants_docker_generator import sls_dockerfile

dockerfile = sls_dockerfile(
    base_image=subsystem.docker_base_image,
    product_name=target.product_name,
    product_version=target.version,
    product_group=target.product_group,
    dist_name=f"{target.product_name}-{target.version}",
    tarball_name=f"{target.product_name}-{target.version}.sls.tgz",
    install_path=subsystem.install_path,
    use_hook_init=bool(target.hooks),
    expose_ports=tuple(detected_ports),
)
```

### sls-publish → release-hub

The `sls-publish` goal creates an `ApolloHubClient` and publishes:

```python
from pants_release_hub import ApolloHubClient, PublishRequest, build_artifact_url

client = ApolloHubClient(
    base_url=subsystem.apollo_hub_url,
    auth_token=subsystem.apollo_auth_token,
)
result = client.publish_release(PublishRequest(
    product_group=manifest.product_group,
    product_name=manifest.product_name,
    product_version=manifest.product_version,
    product_type=manifest.product_type,
    artifact_url=build_artifact_url(
        subsystem.docker_registry,
        manifest.product_name,
        manifest.product_version,
    ),
    manifest_yaml=manifest.content,
    dry_run=publish_subsystem.dry_run,
))
```

## Generated Distribution Layout

```
catalog-service-1.0.0/
  deployment/
    manifest.yml                      ← sls-manifest output
    product-dependencies.lock         ← sls-lock output (if deps exist)
  service/
    bin/
      darwin-amd64/
        python-service-launcher       ← Go binary (from GitHub)
      darwin-arm64/
        python-service-launcher
      linux-amd64/
        python-service-launcher
      linux-arm64/
        python-service-launcher
      catalog-service.pex             ← PEX binary (from Python sources)
    lib/
      hooks.sh                        ← Hook execution library (if hooks)
    monitoring/
      bin/
        check.sh                      ← Health check script
    run/
      init                            ← Service init script
      launcher-static.yml             ← Launcher config (from BUILD fields)
      launcher-check.yml              ← Health check config (if check_args)
  var/
    logs/
    metrics/
    state/
  hooks/                              ← Only if hooks configured
    pre-configure.d/
    configure.d/
    pre-startup.d/
      10-migrate.sh                   ← User hook scripts
    startup.d/
    post-startup.d/
    pre-shutdown.d/
    shutdown.d/
```

## Hook Init System

When `hooks` is set on `sls_service`, the container entrypoint runs hook phases:

```
Container starts
  → entrypoint.sh
    → pre-configure.d/*.sh
      → configure.d/*.sh
        → pre-startup.d/*.sh (e.g., DB migrations)
          → startup.d/*.sh (launches service-launcher)
            → post-startup.d/*.sh (warm caches, etc.)
              → [READY — service accepting traffic]
                → SIGTERM received
                  → pre-shutdown.d/*.sh
                    → shutdown.d/*.sh
                      → [EXIT]
```

## Docker Generation

The generated Dockerfile follows the SLS pattern:

```dockerfile
FROM python:3.11-slim
LABEL org.opencontainers.image.title="catalog-service"
LABEL org.opencontainers.image.version="1.0.0"

COPY catalog-service-1.0.0.sls.tgz /tmp/
RUN tar xzf /tmp/catalog-service-1.0.0.sls.tgz -C /opt/services/ && \
    rm /tmp/catalog-service-1.0.0.sls.tgz

WORKDIR /opt/services/catalog-service-1.0.0
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s \
  CMD /opt/services/catalog-service-1.0.0/service/monitoring/bin/check.sh

ENTRYPOINT ["/opt/services/catalog-service-1.0.0/service/run/init"]
```

If hooks are enabled, `ENTRYPOINT` uses `entrypoint.sh` instead of `init` directly.

## Publishing Flow

```
sls-publish goal
  ├── Read manifest from dist/
  ├── detect_version("1.0.0")
  │     → format=SEMVER, release_type=RELEASE
  ├── default_channel_for_release_type(RELEASE)
  │     → channel="stable"
  ├── build_artifact_url(registry, name, version)
  │     → "oci://registry.example.io/catalog-service:1.0.0"
  └── ApolloHubClient.publish_release(PublishRequest(...))
        → POST /api/v1/releases
        → PublishResult(success=True, release_id="...")
```

## Service Launcher Integration

The Go binary reads `launcher-static.yml` at container startup:

```yaml
# Generated from BUILD target fields
configType: python
launchMode: uvicorn
entryPoint: catalog.app:app
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

The launcher's 11-step sequence:
```
Read config → CPU detection → Memory limits → Create dirs
→ Set rlimits → Build command/env → Fork → PID file
→ Readiness probe → RSS watchdog
```
