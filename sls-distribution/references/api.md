# SLS-Distribution — Core API Reference

> Part of the sls-distribution skill. See [SKILL.md](../SKILL.md) for overview.

## Table of Contents
- [sls_service Target](#sls_service-target)
- [sls_asset Target](#sls_asset-target)
- [sls_product_dependency Target](#sls_product_dependency-target)
- [sls_product_incompatibility Target](#sls_product_incompatibility-target)
- [sls_artifact Target](#sls_artifact-target)
- [Goals](#goals)
- [Subsystem Configuration](#subsystem-configuration)
- [Launcher Subsystem](#launcher-subsystem)
- [Distribution Layout](#distribution-layout)

## sls_service Target

Packages a Python service as an SLS distribution with manifest generation, service layout, init scripts, and Docker support.

### Identity Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `product_group` | str | **REQUIRED** | Maven-style group (e.g., `com.example`). Lowercase letters, digits, dots, hyphens. |
| `product_name` | str | **REQUIRED** | Product name (e.g., `my-service`). Must start with lowercase letter. |
| `version` | str | **REQUIRED** | SLS orderable version: `X.Y.Z`, `X.Y.Z-rcN`, `X.Y.Z-N-gHASH` |
| `product_type` | str | `"helm.v1"` | `helm.v1`, `asset.v1`, or `service.v1` |
| `display_name` | str | None | Human-readable display name |
| `description` | str | None | Product description |

### Python Service Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `entrypoint` | str | **REQUIRED** | `module:callable` format (e.g., `app:app`) |
| `command` | str | `"uvicorn"` | Command to run the service |
| `args` | list[str] | `["--host", "0.0.0.0", "--port", "8080"]` | Arguments passed to command |
| `python_version` | str | `"3.11"` | Python version requirement |
| `env` | dict[str, str] | `{}` | Environment variables |
| `pex_binary` | str | None | PEX binary path (overrides command + entrypoint) |
| `hooks` | dict[str, str] | `{}` | Hook scripts: `"<phase>.d/<name>.sh": "path/to/script"` |

### Health Check Fields (mutually exclusive)

| Field | Type | Description |
|-------|------|-------------|
| `check_args` | list[str] | Reuse launcher binary with different args. Generates `launcher-check.yml` |
| `check_command` | str | Custom command (e.g., `curl -sf localhost:8080/health`). Generates `check.sh` |
| `check_script` | str | Path to custom `check.sh` script (copied verbatim) |

### Kubernetes Resource Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `resource_requests` | dict | `{}` | e.g., `{"cpu": "100m", "memory": "128Mi"}` |
| `resource_limits` | dict | `{}` | e.g., `{"cpu": "500m", "memory": "512Mi"}` |
| `replication_desired` | int | None | Desired replica count |
| `replication_min` | int | None | Minimum replicas. Constraint: `min <= desired` |
| `replication_max` | int | None | Maximum replicas. Constraint: `desired <= max` |

### Metadata Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `labels` | dict | `{}` | Product labels |
| `annotations` | dict | `{}` | Product annotations |
| `traits` | list[str] | `()` | Capability traits (e.g., `["api", "web"]`) |
| `manifest_extensions` | dict | `{}` | Additional manifest YAML keys |
| `product_dependencies` | deps | None | References to `sls_product_dependency` targets |
| `product_incompatibilities` | deps | None | References to `sls_product_incompatibility` targets |
| `artifacts` | deps | None | References to `sls_artifact` targets |

## sls_asset Target

Static file distribution packaged as SLS asset (`product-type: asset.v1`). No runtime, init script, or health checks.

Inherits identity and metadata fields from `sls_service`, plus:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `assets` | dict | `{}` | Source → destination mapping under `asset/` directory |
| `sources` | globs | `("**/*",)` | Files to include (any file type) |

## sls_product_dependency Target

Declares a product dependency.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `product_group` | str | **REQUIRED** | Maven-style group |
| `product_name` | str | **REQUIRED** | Product name |
| `minimum_version` | str | **REQUIRED** | Minimum compatible version (SLS orderable) |
| `maximum_version` | str | None | Maximum version. Defaults to `<major>.x.x` |
| `recommended_version` | str | None | Recommended version |
| `optional` | bool | `False` | Whether optional |

**Constraint**: `minimum_version != maximum_version` (prevents lockstep upgrade antipattern).

## sls_product_incompatibility Target

Declares a product incompatibility.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `product_group` | str | **REQUIRED** | Maven-style group |
| `product_name` | str | **REQUIRED** | Product name |
| `version_range` | str | **REQUIRED** | Incompatible range (e.g., `< 2.0.0`) |
| `reason` | str | **REQUIRED** | Explanation for incompatibility |

## sls_artifact Target

An artifact reference (OCI image, Helm chart) included in manifest extensions.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | str | `"oci"` | Artifact type |
| `uri` | str | **REQUIRED** | Artifact URI |
| `artifact_name` | str | None | Artifact name |
| `digest` | str | None | Content digest (e.g., `sha256:abc123...`) |

## Goals

| Goal | Description |
|------|-------------|
| `sls-manifest` | Generate `deployment/manifest.yml` |
| `sls-validate` | Validate manifests against schema + semantic rules |
| `sls-package` | Package into `.sls.tgz` tarball with full layout |
| `sls-docker` | Generate Dockerfile + .dockerignore |
| `sls-lock` | Generate `product-dependencies.lock` |
| `sls-publish` | Publish to Apollo Hub (supports `--dry-run`) |

```bash
pants sls-manifest sls-validate sls-package sls-docker src/services:my-service
pants sls-publish --dry-run src/services:my-service
pants sls-publish src/services:my-service
```

## Subsystem Configuration

**Scope**: `[sls-distribution]` in `pants.toml`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `default_python_version` | str | `"3.11"` | Default Python version |
| `default_command` | str | `"uvicorn"` | Default service command |
| `docker_base_image` | str | `"python:3.11-slim"` | Base Docker image |
| `docker_registry` | str | `""` | Docker registry URL |
| `apollo_hub_url` | str | `""` | Apollo Hub URL for publishing |
| `apollo_auth_token` | str | `""` | Bearer token (also reads `APOLLO_AUTH_TOKEN` env) |
| `publish_channel` | str | `""` | Override channel (empty = auto-detect from version) |
| `install_path` | str | `"/opt/services"` | Install path in Docker container |
| `manifest_version` | str | `"1.0"` | Manifest version string |
| `strict_validation` | bool | `True` | Enable JSON schema validation |

## Launcher Subsystem

**Scope**: `[python-service-launcher]` in `pants.toml`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `version` | str | `"v0.1.0"` | GitHub release tag to download |
| `github_repo` | str | `"jaymd96/python-service-launcher"` | Repository hosting binaries |
| `known_versions` | list | SHA256 hashes for 4 platforms | `version|os|arch|sha256|size` entries |

Supported platforms: `darwin-amd64`, `darwin-arm64`, `linux-amd64`, `linux-arm64`

## Distribution Layout

```
<product-name>-<version>/
  deployment/
    manifest.yml
    product-dependencies.lock     (if dependencies exist)
  service/
    bin/
      <os>-<arch>/
        python-service-launcher   (4 platform binaries)
      <product-name>.pex
    lib/
      hooks.sh                    (if hooks enabled)
    monitoring/
      bin/
        check.sh                  (if health check configured)
    run/
      init                        (service init script)
      launcher.yml
      launcher-check.yml          (if check_args mode)
  var/
    logs/
    metrics/
    state/
  hooks/                          (if hooks enabled)
    pre-configure.d/
    configure.d/
    pre-startup.d/
    startup.d/
    post-startup.d/
    pre-shutdown.d/
    shutdown.d/
```
