# Docker-Generator — Examples & Gotchas

> Part of the docker-generator skill. See [SKILL.md](../SKILL.md) for overview.

## Basic Dockerfile

```python
from pants_docker_generator import DockerfileBuilder

dockerfile = (
    DockerfileBuilder()
    .from_("python:3.11-slim", alias="runtime")
    .workdir("/app")
    .copy("requirements.txt", ".")
    .run("pip install --no-cache-dir -r requirements.txt")
    .copy(".", ".")
    .expose(8080)
    .entrypoint("python", "-m", "app")
    .build()
)
print(dockerfile.render())
```

## Multi-Stage Build

```python
from pants_docker_generator import DockerfileBuilder

builder = (
    DockerfileBuilder()
    # Build stage
    .from_("python:3.11", alias="build")
    .workdir("/build")
    .copy("requirements.txt", ".")
    .run("pip wheel --no-deps -w /wheels -r requirements.txt")
    .copy(".", ".")
    .run("pip wheel --no-deps -w /wheels .")
    .blank()
    # Runtime stage
    .from_("python:3.11-slim", alias="runtime")
    .copy("/wheels", "/wheels", from_stage="build")
    .run("pip install --no-cache-dir /wheels/*")
    .user("nobody")
    .expose(8080)
    .entrypoint("python", "-m", "app")
    .build()
)
```

## SLS Distribution Dockerfile

```python
from pants_docker_generator import sls_dockerfile

df = sls_dockerfile(
    base_image="python:3.11-slim",
    product_name="catalog-service",
    product_version="1.0.0",
    product_group="com.example",
    dist_name="catalog-service-1.0.0",
    tarball_name="catalog-service-1.0.0.sls.tgz",
    expose_ports=(8080,),
    use_hook_init=True,
)
print(df.render())
```

## Using Pre-Built Directives

```python
from pants_docker_generator import DockerfileBuilder, HealthCheck, Label, oci_labels

dockerfile = (
    DockerfileBuilder()
    .from_("python:3.11-slim")
    .directive(oci_labels(title="My App", version="1.0.0", vendor="Example"))
    .workdir("/app")
    .copy(".", ".")
    .directive(HealthCheck(
        command="curl -f http://localhost:8080/health || exit 1",
        interval_seconds=30,
        timeout_seconds=10,
        retries=5,
    ))
    .entrypoint("python", "app.py")
    .build()
)
```

## Custom Dockerignore

```python
from pants_docker_generator import DockerignoreConfig, generate_dockerignore

# Simple helper
content = generate_dockerignore(
    allow=("app/", "requirements.txt", "config/"),
    comment="Auto-generated for my-service",
)

# Full control
config = DockerignoreConfig(
    ignore_all=True,
    allow_patterns=("app/", "requirements.txt"),
    deny_patterns=("**/__pycache__", "**/*.pyc"),
    comment="Custom ignore",
)
print(config.render())
```

## Platform-Specific Build

```python
dockerfile = (
    DockerfileBuilder()
    .from_("python:3.11-slim", alias="runtime", platform="linux/amd64")
    .workdir("/app")
    .copy(".", ".")
    .entrypoint("python", "app.py")
    .build()
)
```

## Gotchas

1. **Zero external dependencies**: This library has no dependencies outside the Python standard library. It generates Dockerfile text — it does not execute Docker commands
2. **Frozen dataclasses throughout**: All directives, stages, and Dockerfile objects are immutable. The builder is the only mutable interface
3. **Exec form for ENTRYPOINT/CMD**: `entrypoint()` and `cmd()` always render as JSON array (exec form), never shell form. This ensures proper signal handling in containers
4. **`from_()` not `from()`**: The method is `from_()` with a trailing underscore because `from` is a Python reserved word
5. **Stage references use aliases**: When using `copy(from_stage="build")`, the stage alias must match the `alias` parameter from a prior `from_()` call
6. **`build()` is terminal**: After calling `build()`, the builder returns a `Dockerfile` object. You cannot continue chaining after `build()`
7. **Label renders multiline**: When `Label` has multiple entries, it renders as a multiline `LABEL` instruction with backslash continuation
