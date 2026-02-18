# SLS-Distribution — Pipeline & Lifecycle

> Part of the sls-distribution skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [7-Stage Pipeline](#7-stage-pipeline)
- [Stage 1: Setup](#stage-1-setup)
- [Stage 2: Define](#stage-2-define)
- [Stage 3: Manifest](#stage-3-manifest)
- [Stage 4: Validate](#stage-4-validate)
- [Stage 5: Package](#stage-5-package)
- [Stage 6: Docker](#stage-6-docker)
- [Stage 7: Publish](#stage-7-publish)
- [Hook Init Lifecycle](#hook-init-lifecycle)
- [Version Auto-Detection](#version-auto-detection)
- [Validation Rules](#validation-rules)

## 7-Stage Pipeline

```
1. Setup (pants.toml)
   → 2. Define (BUILD sls_service)
     → 3. Manifest (sls-manifest)
       → 4. Validate (sls-validate)
         → 5. Package (sls-package → .sls.tgz)
           → 6. Docker (sls-docker → Dockerfile)
             → 7. Publish (sls-publish → Apollo Hub)
```

Orchestrates three tools: **docker-generator** (Dockerfile), **service-launcher** (Go binary), **release-hub** (Apollo Hub client).

## Stage 1: Setup

Configure the plugin in `pants.toml`:

```toml
[GLOBAL]
backend_packages = ["pants_sls_distribution"]

[sls-distribution]
docker_base_image = "python:3.11-slim"
docker_registry = "registry.example.io"
apollo_hub_url = "https://hub.example.com"
apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"

[python-service-launcher]
version = "v0.1.0"
```

## Stage 2: Define

Create `sls_service` targets in BUILD files:

```python
sls_service(
    name="my-service",
    product_group="com.example",
    product_name="my-service",
    version="1.0.0",
    entrypoint="app:app",
)
```

## Stage 3: Manifest

```bash
pants sls-manifest src/services:my-service
```

Generates `deployment/manifest.yml` with product identity, resource requirements, dependencies, and metadata. Output in `dist/<product-name>/deployment/manifest.yml`.

**Manifest structure**:
```yaml
manifest-version: "1.0"
product-type: helm.v1
product-group: com.example
product-name: my-service
product-version: 1.0.0
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits: { cpu: 500m, memory: 512Mi }
extensions:
  product-dependencies: [...]
  artifacts: [...]
```

## Stage 4: Validate

```bash
pants sls-validate src/services:my-service
```

Runs two validation layers:
1. **Schema validation**: JSON schema (requires `jsonschema` package, controlled by `strict_validation`)
2. **Semantic validation**: product identity format, replication constraints (`min <= desired <= max`), dependency constraints, lockstep upgrade detection, version format

Exit code 0 = valid, 1 = failed.

## Stage 5: Package

```bash
pants sls-package src/services:my-service
```

Creates the full distribution layout in `dist/<dist-name>/` and packages it into `dist/<dist-name>.sls.tgz`. Includes:
- Manifest and lock files
- Service launcher binaries (4 platform variants)
- PEX binary
- Init script, launcher config
- Health check scripts (if configured)
- Hook scripts (if configured)

## Stage 6: Docker

```bash
pants sls-docker src/services:my-service
```

Generates `dist/<dist-name>/docker/Dockerfile` and `.dockerignore` using **docker-generator**. Auto-detects:
- Health check configuration mode
- Hook init system enablement
- Multi-platform support

If hooks are enabled, also generates `docker/hooks/entrypoint.sh` and `docker/hooks/hooks.sh`.

## Stage 7: Publish

```bash
pants sls-publish src/services:my-service
pants sls-publish --dry-run src/services:my-service
```

Publishes to Apollo Hub via **release-hub** client. Requires `apollo_hub_url` and auth token. Channel auto-detected from version format or overridden with `publish_channel`.

## Hook Init Lifecycle

When hooks are configured on an `sls_service`, the hook init system provides an ordered lifecycle:

```
pre-configure.d/  →  configure.d/  →  pre-startup.d/
    →  startup.d/ (starts main service)
        →  post-startup.d/ (service may be running)
            →  [READY]
                →  pre-shutdown.d/  →  shutdown.d/  →  [EXIT]
```

### Hook path format

Keys in the `hooks` field follow `<phase>.d/<script-name>.sh`:

```python
sls_service(
    hooks={
        "pre-startup.d/10-migrate.sh": "scripts/migrate.sh",
        "post-startup.d/20-warm-cache.sh": "scripts/warm_cache.sh",
    },
)
```

Scripts are numbered for ordering (e.g., `10-`, `20-`). All scripts must be POSIX `sh` compatible.

### Execution modes

The embedded `hooks.sh` library provides three execution modes:

| Function | Behavior |
|----------|----------|
| `run_hooks` | Run all scripts in phase, halt on failure |
| `run_hooks_timed` | Same, but emit JSON timing metrics |
| `run_hooks_warn` | Run all, log failures but continue |

### Docker integration

When hooks are enabled, the Docker image uses `entrypoint.sh` as its `ENTRYPOINT` instead of directly invoking the service launcher. The entrypoint runs hook phases in order.

## Version Auto-Detection

Version format determines the publish channel:

| Format | Example | Release Type | Channel |
|--------|---------|-------------|---------|
| `X.Y.Z` | `1.0.0` | RELEASE | `stable` |
| `X.Y.Z-rcN` | `1.0.0-rc1` | RELEASE_CANDIDATE | `beta` |
| `X.Y.Z-N-gHASH` | `1.0.0-5-gabcdef` | SNAPSHOT | `dev` |
| `X.Y.Z-rcN-N-gHASH` | `1.0.0-rc1-5-gabcdef` | RC_SNAPSHOT | `dev` |

Override with `[sls-distribution].publish_channel` in `pants.toml`.

## Validation Rules

### Product identity
- `product_group`: lowercase letters, digits, dots, hyphens (`^[a-z0-9.-]+$`)
- `product_name`: starts with lowercase letter (`^[a-z][a-z0-9.-]*$`)
- `version`: must match SLS orderable format

### Replication constraints
- If set: `min <= desired <= max`
- All values must be positive integers

### Dependency constraints
- No duplicate product dependencies
- `minimum_version` must be SLS orderable
- `minimum_version != maximum_version` (prevents lockstep upgrades)
- `maximum_version` allows `x` wildcards (e.g., `1.x.x`)

### Health check mutual exclusivity
Only one of `check_args`, `check_command`, `check_script` may be set.
