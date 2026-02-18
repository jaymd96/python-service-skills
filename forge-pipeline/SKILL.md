---
name: forge-pipeline
description: Workflow for packaging, containerizing, and publishing Python services using Forge tools. Covers sls-distribution Pants plugin (sls_service targets, 7-stage pipeline), docker-generator (Dockerfile generation), release-hub (Apollo Hub publishing), and service-launcher (Go process manager). Use when packaging a service for deployment. Triggers on forge, sls pipeline, package service, docker build, publish release, deployment pipeline, sls-package, sls-docker, sls-publish.
---

# Forge Pipeline — Package, Containerize, Publish

## Workflow Overview

```
1. Configure Pants (pants.toml)
   → 2. Define target (BUILD sls_service)
     → 3. Generate manifest (sls-manifest)
       → 4. Validate (sls-validate)
         → 5. Package (sls-package → .sls.tgz)
           → 6. Containerize (sls-docker → Dockerfile)
             → 7. Publish (sls-publish → Apollo Hub)
```

## Quick Start

```toml
# pants.toml
[GLOBAL]
backend_packages = ["pants_sls_distribution"]

[sls-distribution]
docker_base_image = "python:3.11-slim"
docker_registry = "registry.example.io"
apollo_hub_url = "https://hub.example.com"
apollo_auth_token = "@env(APOLLO_AUTH_TOKEN)"
```

```python
# BUILD
sls_service(
    name="catalog-service",
    product_group="com.example",
    product_name="catalog-service",
    version="1.0.0",
    entrypoint="catalog.app:app",
)
```

```bash
# Full pipeline
pants sls-validate sls-package sls-docker catalog-service:
pants sls-publish catalog-service:

# Or step by step
pants sls-manifest catalog-service:     # Generate manifest
pants sls-validate catalog-service:     # Validate
pants sls-package catalog-service:      # Package to .sls.tgz
pants sls-docker catalog-service:       # Generate Dockerfile
pants sls-publish catalog-service:      # Publish to Apollo Hub
pants sls-publish --dry-run catalog-service:  # Preview
```

## Tool Integration

```
sls-distribution (Pants plugin — orchestrator)
  ├── docker-generator  → generates Dockerfile + .dockerignore
  ├── service-launcher  → Go binary bundled into distribution
  └── release-hub       → publishes to Apollo Hub
```

- **sls-distribution** defines the `sls_service` target and 7 goals
- **docker-generator** provides `sls_dockerfile()` preset called by `sls-docker`
- **service-launcher** binary is downloaded (SHA256-verified) and bundled by `sls-package`
- **release-hub** provides `ApolloHubClient` used by `sls-publish`

## Key Decisions

### Health check mode

```python
# Option 1: Reuse launcher (recommended)
sls_service(check_args=["--check"], ...)

# Option 2: Custom command
sls_service(check_command="curl -sf localhost:8080/health", ...)

# Option 3: Custom script
sls_service(check_script="scripts/check.sh", ...)
```

### Version → channel mapping

| Version | Channel | Example |
|---------|---------|---------|
| `X.Y.Z` | `stable` | `1.0.0` |
| `X.Y.Z-rcN` | `beta` | `1.0.0-rc1` |
| `X.Y.Z-N-gHASH` | `dev` | `1.0.0-5-gabcdef` |

## References

- **[pipeline.md](references/pipeline.md)** — Detailed pipeline stages, tool interactions, generated outputs
- **[configuration.md](references/configuration.md)** — Complete configuration reference, BUILD examples, CI integration, gotchas

## Grep Patterns

- `sls_service\(|sls_asset\(` — Find SLS target definitions
- `sls-distribution|backend_packages.*sls` — Find Pants configuration
- `sls-package|sls-docker|sls-publish` — Find pipeline goal invocations
- `DockerfileBuilder|sls_dockerfile` — Find Dockerfile generation
- `ApolloHubClient|publish_release` — Find release publishing
