---
name: sls-distribution
description: Pants plugin for packaging Python services as SLS distributions. Use when defining sls_service() BUILD targets, running sls-manifest/sls-validate/sls-package/sls-docker/sls-publish goals, configuring hook init system, health checks, or resource limits. Orchestrates docker-generator, release-hub, and service-launcher. Triggers on sls-distribution, sls_service, pants, sls-package, sls-docker, sls-publish, deployment pipeline.
---

# sls-distribution — SLS Packaging Pipeline (v0.1.0)

## Quick Start

**pants.toml**:
```toml
[GLOBAL]
backend_packages = ["pants_sls_distribution"]

[sls-distribution]
docker_base_image = "python:3.11-slim"
docker_registry = "registry.example.io"
apollo_hub_url = "https://hub.example.com"
apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"
```

**BUILD**:
```python
sls_service(
    name="my-service",
    product_group="com.example",
    product_name="my-service",
    version="1.0.0",
    entrypoint="app:app",
)
```

```bash
pants sls-validate sls-package sls-docker my-service:
pants sls-publish my-service:
```

## Key Patterns

### 7-stage pipeline
```
1. Setup (pants.toml)  ->  2. Define (BUILD sls_service)  ->  3. Manifest
->  4. Validate  ->  5. Package (.sls.tgz)  ->  6. Docker  ->  7. Publish
```

### Health check modes (mutually exclusive)
```python
sls_service(check_args=["--check"])               # Reuses launcher binary
sls_service(check_command="curl -sf localhost:8080/health")  # Custom command
sls_service(check_script="scripts/check.sh")       # Custom script
```

## References

- **[api.md](references/api.md)** — sls_service target fields, goals, subsystem options, layout structure
- **[pipeline.md](references/pipeline.md)** — 7-stage pipeline details, hook init lifecycle, version auto-detection
- **[examples.md](references/examples.md)** — Complete BUILD examples, configuration reference, gotchas

## Grep Patterns

- `sls_service\(|sls_asset\(|sls_product_dependency\(` — Find SLS target definitions
- `sls-distribution|backend_packages.*sls` — Find Pants configuration
- `sls-package|sls-docker|sls-publish` — Find pipeline goal invocations
