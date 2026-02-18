# Forge Pipeline — Configuration & Examples

> Part of the forge-pipeline workflow. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [Complete BUILD Example](#complete-build-example)
- [pants.toml Configuration](#pantstoml-configuration)
- [CI Pipeline Integration](#ci-pipeline-integration)
- [Multi-Service Monorepo](#multi-service-monorepo)
- [Asset Distribution](#asset-distribution)
- [Custom Dockerfile Override](#custom-dockerfile-override)
- [Gotchas](#gotchas)

## Complete BUILD Example

```python
# src/services/catalog/BUILD

# Main service target
sls_service(
    name="catalog-service",
    product_group="com.example",
    product_name="catalog-service",
    version="1.0.0",
    display_name="Catalog Service",
    description="Product catalog API for Apollo platform",
    entrypoint="catalog.app:app",
    command="uvicorn",
    args=["--host", "0.0.0.0", "--port", "8080", "--workers", "4"],
    python_version="3.11",
    env={
        "LOG_LEVEL": "INFO",
        "DB_POOL_SIZE": "10",
        "OTEL_EXPORTER_OTLP_ENDPOINT": "http://otel-collector:4317",
    },
    check_args=["--check"],
    resource_requests={"cpu": "100m", "memory": "256Mi"},
    resource_limits={"cpu": "500m", "memory": "512Mi"},
    replication_desired=2,
    replication_min=1,
    replication_max=5,
    labels={"team": "platform", "tier": "api"},
    annotations={"owner": "platform@team.com"},
    traits=["api", "web"],
    product_dependencies=[":database-dep", ":auth-dep"],
    hooks={
        "pre-startup.d/10-migrate.sh": "scripts/migrate.sh",
        "post-startup.d/20-warm-cache.sh": "scripts/warm_cache.sh",
    },
)

# Dependencies
sls_product_dependency(
    name="database-dep",
    product_group="com.example",
    product_name="postgres",
    minimum_version="14.0.0",
    maximum_version="16.x.x",
    recommended_version="15.5.0",
)

sls_product_dependency(
    name="auth-dep",
    product_group="com.example",
    product_name="auth-service",
    minimum_version="2.0.0",
    optional=True,
)

# Incompatibilities
sls_product_incompatibility(
    name="legacy-incompat",
    product_group="com.legacy",
    product_name="old-catalog",
    version_range="< 3.0.0",
    reason="Incompatible data format after v3 migration",
)

# Artifacts
sls_artifact(
    name="docker-image",
    type="oci",
    uri="registry.example.io/catalog-service:1.0.0",
)
```

## pants.toml Configuration

```toml
[GLOBAL]
backend_packages = [
    "pants_sls_distribution",
]

# SLS Distribution plugin
[sls-distribution]
default_python_version = "3.11"
default_command = "uvicorn"
docker_base_image = "python:3.11-slim"
docker_registry = "registry.example.io"
apollo_hub_url = "https://hub.example.com"
apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"
install_path = "/opt/services"
manifest_version = "1.0"
strict_validation = true
# publish_channel = ""  # Empty = auto-detect from version

# Service launcher binary
[python-service-launcher]
version = "v0.1.0"
github_repo = "jaymd96/python-service-launcher"
```

## CI Pipeline Integration

```bash
#!/bin/bash
# ci/deploy.sh — CI pipeline script

set -euo pipefail

VERSION="${1:?Usage: deploy.sh <version>}"

echo "=== Step 1: Validate ==="
pants sls-validate src/services/catalog:catalog-service

echo "=== Step 2: Package ==="
pants sls-package src/services/catalog:catalog-service

echo "=== Step 3: Docker ==="
pants sls-docker src/services/catalog:catalog-service

echo "=== Step 4: Build Docker image ==="
cd dist/catalog-service-${VERSION}/docker
docker build -t registry.example.io/catalog-service:${VERSION} .
docker push registry.example.io/catalog-service:${VERSION}

echo "=== Step 5: Publish to Apollo Hub ==="
pants sls-publish src/services/catalog:catalog-service

echo "=== Done ==="
```

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    tags: ["v*"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pantsbuild/actions/init-pants@v6

      - name: Validate & Package
        run: |
          pants sls-validate sls-package sls-docker ::

      - name: Build & Push Docker
        run: |
          docker build -t $REGISTRY/$SERVICE:$VERSION -f dist/*/docker/Dockerfile .
          docker push $REGISTRY/$SERVICE:$VERSION

      - name: Publish to Apollo Hub
        env:
          APOLLO_AUTH_TOKEN: ${{ secrets.APOLLO_AUTH_TOKEN }}
        run: pants sls-publish ::
```

## Multi-Service Monorepo

```
monorepo/
  pants.toml
  src/
    services/
      catalog/
        BUILD                    # sls_service(name="catalog-service", ...)
        catalog/
          app.py
      auth/
        BUILD                    # sls_service(name="auth-service", ...)
        auth/
          app.py
      gateway/
        BUILD                    # sls_service(name="api-gateway", ...)
        gateway/
          app.py
```

```bash
# Pipeline for all services
pants sls-validate sls-package sls-docker ::

# Pipeline for specific service
pants sls-validate sls-package sls-docker src/services/catalog:

# Publish all
pants sls-publish ::
```

## Asset Distribution

```python
# src/assets/web/BUILD
sls_asset(
    name="web-assets",
    product_group="com.example",
    product_name="web-assets",
    version="1.0.0",
    assets={
        "static/css/": "web/css/",
        "static/js/": "web/js/",
        "static/images/": "web/images/",
        "config/nginx.conf": "conf/nginx.conf",
    },
    labels={"team": "frontend"},
)
```

## Custom Dockerfile Override

If you need a Dockerfile beyond what `sls-docker` generates, use `docker-generator` directly:

```python
# scripts/build_custom_docker.py
from pants_docker_generator import DockerfileBuilder, sls_dockerfile, oci_labels

# Start with SLS preset, then customize
base = sls_dockerfile(
    base_image="python:3.11-slim",
    product_name="catalog-service",
    product_version="1.0.0",
    product_group="com.example",
    dist_name="catalog-service-1.0.0",
    tarball_name="catalog-service-1.0.0.sls.tgz",
)

# Or build from scratch with full control
dockerfile = (
    DockerfileBuilder()
    .from_("python:3.11-slim", alias="runtime")
    .directive(oci_labels(title="catalog-service", version="1.0.0"))
    .run("apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*")
    .copy("catalog-service-1.0.0.sls.tgz", "/tmp/")
    .run("tar xzf /tmp/catalog-service-1.0.0.sls.tgz -C /opt/services/ && rm /tmp/*.tgz")
    .workdir("/opt/services/catalog-service-1.0.0")
    .expose(8080)
    .healthcheck("curl -sf http://localhost:8080/status/health || exit 1",
                 interval_seconds=15, timeout_seconds=5, retries=3)
    .user("nobody")
    .entrypoint("service/run/init")
    .build()
)
print(dockerfile.render())
```

## Gotchas

1. **Version must be SLS orderable**: Only `X.Y.Z`, `X.Y.Z-rcN`, `X.Y.Z-N-gHASH` formats. PEP 440 (`1.0.0a1`) and CalVer (`2024.01.15`) are rejected by `sls-validate`
2. **Health check modes are mutually exclusive**: Only one of `check_args`, `check_command`, `check_script`. Setting multiple causes validation failure
3. **`product_name` must start with lowercase letter**: `My-Service` is rejected, `my-service` is accepted
4. **Lockstep upgrades blocked**: `minimum_version == maximum_version` on dependencies is rejected — this prevents forced simultaneous upgrades
5. **Auth token supports `@env()`**: Use `apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"` in pants.toml for environment variable expansion
6. **Hook scripts must be POSIX sh**: Use `#!/bin/sh`, not `#!/bin/bash`. Bash may not be available in slim Docker images
7. **Launcher binaries are SHA256-verified**: If hashes don't match the `known_versions` in pants.toml, the build fails. Update hashes when upgrading launcher version
8. **`sls-publish --dry-run` doesn't build Docker images**: It only previews the Apollo Hub publish request. Docker build and push is a separate step
9. **Order of goals matters**: Run `sls-validate` before `sls-package`. Run `sls-package` before `sls-docker` (Docker needs the tarball). Run `sls-docker` before `sls-publish` (publish needs the artifact URL)
10. **Auto-channel detection order**: `detect_version()` tries CalVer → SLS → PEP 440 → SemVer. A version like `2024.1.0` matches CalVer first, mapping to `stable` channel
