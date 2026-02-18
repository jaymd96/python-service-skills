# SLS-Distribution — Examples & Gotchas

> Part of the sls-distribution skill. See [SKILL.md](../SKILL.md) for overview.

## Complete Service Definition

```python
# src/services/BUILD
sls_service(
    name="catalog-service",
    product_group="com.example",
    product_name="catalog-service",
    version="1.0.0",
    display_name="Catalog Service",
    description="Product catalog API",
    entrypoint="catalog.app:app",
    command="uvicorn",
    args=["--host", "0.0.0.0", "--port", "8080", "--workers", "4"],
    python_version="3.11",
    env={"LOG_LEVEL": "INFO", "DB_POOL_SIZE": "10"},
    check_args=["--check"],
    resource_requests={"cpu": "100m", "memory": "128Mi"},
    resource_limits={"cpu": "500m", "memory": "512Mi"},
    replication_desired=2,
    replication_min=1,
    replication_max=5,
    labels={"team": "platform", "tier": "api"},
    traits=["api", "web"],
    product_dependencies=[":database-dep"],
    hooks={
        "pre-startup.d/10-migrate.sh": "scripts/migrate.sh",
        "post-startup.d/20-warm-cache.sh": "scripts/warm_cache.sh",
    },
)

sls_product_dependency(
    name="database-dep",
    product_group="com.example",
    product_name="sample-database",
    minimum_version="1.0.0",
    maximum_version="2.x.x",
    recommended_version="1.5.0",
)
```

## Minimal Service

```python
sls_service(
    name="my-service",
    product_group="com.example",
    product_name="my-service",
    version="1.0.0",
    entrypoint="app:app",
)
```

## Asset Distribution

```python
sls_asset(
    name="web-assets",
    product_group="com.example",
    product_name="web-assets",
    version="1.0.0",
    assets={
        "static/": "web/static/",
        "config/": "conf/",
    },
)
```

## Health Check Modes

```python
# Mode 1: Reuse launcher binary (Palantir pattern)
sls_service(
    name="with-launcher-check",
    check_args=["--check"],
    # ...
)

# Mode 2: Custom command
sls_service(
    name="with-custom-check",
    check_command="curl -sf localhost:8080/health",
    # ...
)

# Mode 3: Custom script
sls_service(
    name="with-script-check",
    check_script="scripts/check.sh",
    # ...
)
```

## pants.toml Configuration

```toml
[GLOBAL]
backend_packages = ["pants_sls_distribution"]

[sls-distribution]
docker_base_image = "python:3.11-slim"
docker_registry = "registry.example.io"
apollo_hub_url = "https://hub.example.com"
apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"
install_path = "/opt/services"
strict_validation = true

[python-service-launcher]
version = "v0.1.0"
github_repo = "jaymd96/python-service-launcher"
```

## Pipeline Commands

```bash
# Full pipeline
pants sls-manifest sls-validate sls-package sls-docker src/services:

# Individual stages
pants sls-manifest src/services:my-service
pants sls-validate src/services:my-service
pants sls-package src/services:my-service
pants sls-docker src/services:my-service

# Publish
pants sls-publish --dry-run src/services:my-service   # Preview
pants sls-publish src/services:my-service              # Publish to Apollo Hub

# Lock dependencies
pants sls-lock src/services:my-service
```

## Hook Scripts

```bash
# scripts/migrate.sh — pre-startup hook
#!/bin/sh
set -e
echo "Running database migrations..."
python -m alembic upgrade head
echo "Migrations complete."
```

```python
# BUILD — reference hook scripts
sls_service(
    name="my-service",
    hooks={
        "pre-startup.d/10-migrate.sh": "scripts/migrate.sh",
    },
    # ...
)
```

## Gotchas

1. **Health check modes are mutually exclusive**: Only one of `check_args`, `check_command`, `check_script` can be set. Setting multiple causes validation failure
2. **Version must be SLS orderable**: Only `X.Y.Z`, `X.Y.Z-rcN`, `X.Y.Z-N-gHASH`, `X.Y.Z-rcN-N-gHASH` are valid. PEP 440 and CalVer are not accepted
3. **`product_name` must start with a lowercase letter**: The regex is `^[a-z][a-z0-9.-]*$`. Starting with a digit is rejected
4. **Lockstep upgrades are rejected**: `minimum_version` cannot equal `maximum_version` on dependencies — this prevents forced simultaneous upgrades
5. **Replication constraints are validated**: If any replication field is set, the relationship `min <= desired <= max` must hold
6. **Hook scripts must be POSIX sh**: Not bash — use `#!/bin/sh` and avoid bashisms
7. **Auth token resolution**: `apollo_auth_token` in pants.toml supports `@env(VAR_NAME)` syntax, or set `APOLLO_AUTH_TOKEN` environment variable directly
8. **Launcher binaries are cross-compiled**: Four platform variants are bundled (darwin/linux × amd64/arm64). The correct binary is selected at runtime
